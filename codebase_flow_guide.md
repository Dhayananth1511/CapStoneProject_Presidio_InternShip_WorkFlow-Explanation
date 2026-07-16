# End-to-End Codebase Flow & Architectural Document

This document traces the complete lifecycle of request processing, agent interaction, and external tool execution within the Agentic Travel Planner application. It bridges the gap between the React frontend pages and the backend multi-agent system.

---

## 🗺️ Architectural Topology & Sequence Flow

Below is an ASCII sequence map showing the journey of a trip planning task from the client to the backend swarm and database, all the way to final booking.

```
+------------+            +------------+            +----------------+            +-------------------+
|  Traveler  |            |  Express   |            | PlannerSwarm   |            |   Parallel APIs   |
| (Frontend) |            | Controller |            |   Supervisor   |            |   & MCP Servers   |
+-----+------+            +-----+------+            +-------+--------+            +---------+---------+
      |                         |                           |                               |
      | 1. Chat input           |                           |                               |
      +------------------------>|                           |                               |
      |    (POST /plan)         | 2. planTrip()             |                               |
      |                         +-------------------------->|                               |
      |                         |                           | 3. Extract & Validate Slots   |
      |                         |                           +------------------             |
      |                         |                           |                 |             |
      |                         |                           |<----------------+             |
      |                         |                           |                               |
      |                         |                           | 4. Fetch Weather/Stay/Transit |
      |                         |                           +------------------------------>|
      |                         |                           |    (Concurrent invocation)    |
      |                         |                           |                               |
      |                         |                           | 5. Aggregate response         |
      |                         |                           |<------------------------------|
      |                         |                           |                               |
      |                         |                           | 6. Check Budget Feasibility   |
      |                         |                           +------------------             |
      |                         |                           |                 |             |
      |                         |                           |<----------------+             |
      |                         |                           |                               |
      |                         |                           | 7. Batch Day Iterations (Itinerary)
      |                         |                           +------------------             |
      |                         |                           |                 |             |
      |                         |                           |<----------------+             |
      |                         |                           |                               |
      |                         |                           | 8. Local Commute Routing      |
      |                         |                           +------------------------------>|
      |                         |                           |    (Geoapify directions tool) |
      |                         |                           |                               |
      |                         |                           | 9. Enrich and cap budget      |
      |                         |                           |<------------------------------|
      |                         |                           |                               |
      |                         |                           | 10. Synthesize Markdown       |
      |                         |                           +------------------             |
      |                         |                           |                 |             |
      |                         |                           |<----------------+             |
      |                         | 11. Save DRAFT/PLANNED    |                               |
      |                         |<--------------------------+                               |
      | 12. Context + Markdown  |                           |                               |
      |<------------------------+                           |                               |
      |                                                     |                               |
      | ======================= INTERACTIVE CUSTOMIZATION STAGE =========================== |
      |                                                     |                               |
      | 13. Customize Stay/Transport/Replans                |                               |
      +---------------------------------------------------->|                               |
      |                                                     | Reruns target filters, preserves cache
      |                                                     +-------------------------------+
      |                                                                                     |
      | ============================ APPROVAL & FINAL BOOKING ============================= |
      |                                                                                     |
      | 14. Confirm & Approve Click (POST /approve)                                         |
      +------------------------>|                                                           |
      |                         | 15. Create Google Calendar Events                         |
      |                         +---------------------------------------------------------->|
      |                         | 16. Confirmed response + status CONFIRMED                 |
      |<------------------------+                                                           |
```

---

## 🚀 Step 1: Frontend User Engagement

### 1. Planning Request Creation
- **Trigger**: The traveler submits a message on [ChatPage.tsx](file:///d:/Presidio%20Capstone%20Project/client/src/pages/ChatPage.tsx) (e.g. *"Plan a 4-day historical architecture sight-seeing trip from Chennai to Hyderabad starting on 2026-08-10 with a budget of ₹45,000"*).
- **Client Processing**:
  - The message is validated.
  - If a planning session is ongoing, the existing `sessionId` (representing the tripId in the URL) is attached.
  - An HTTP `POST` request is fired to the backend endpoint: `/api/trips/plan` with payload: `{ message, tripId }`.

---

## 🪵 Step 2: Backend HTTP Routes & Controller Guarding

The request reaches [tripRoutes.ts](file:///d:/Presidio%20Capstone%20Project/server/src/routes/tripRoutes.ts) and is handled by the controllers:

### 1. Guarding & Authentication
- **Token Check**: The request passes through the `authenticate` middleware, which parses the Bearer JWT and sets `req.user`.
- **Role Verification**: Per compliance rules, only **Travelers** (not admin accounts) are permitted to write or request trip calculations.

### 2. Controller Validation ([tripController.ts](file:///d:/Presidio%20Capstone%20Project/server/src/controllers/tripController.ts))
- **Parameters**: `createOrUpdateTrip` checks input parameters.
- **Safety check**: It passes the user input message through `isMessageSafe` (preventing system prompt injection strings).
- **Trip Status Check**: If an existing `tripId` is submitted, the controller checks MongoDB to verify that the trip status is not already `CONFIRMED`. If it is confirmed, the request is rejected immediately.
- **Service Delegation**: The arguments are forwarded to the service layer by calling: `planTrip(message, userId, tripId)`.

---

## 🧠 Step 3: Service Layer Context Hydration ([plannerService.ts](file:///d:/Presidio%20Capstone%20Project/server/src/services/plannerService.ts))

The planner service is responsible for loading the execution state, managing memory, and persisting the resulting context.

### 1. Profile Memory Retrieval
- Fetches the user model from MongoDB.
- Retrieves the **Long-Term Memory** string, which contains previous traveler preferences and destinations.

### 2. Context Initialization
- Decodes the `existingTripId`:
  - If found: Rebuilds the schema layout into `TripContext` (carrying forward retrieved weather, transport, accommodation, activities, itineraries).
  - If not found: Generates a brand new UUID for `sessionId` and constructs a clean context sheet.
- Pushes the user message into `context.conversationHistory`.

### 3. Agent Delegation
- Calls `runPlannerAgent(userMessage, context, longTermMemory)`.

---

## 👑 Step 4: The Supervisor Agent Swarm Route ([plannerAgent.ts](file:///d:/Presidio%20Capstone%20Project/server/src/agents/plannerAgent.ts))

The Supervisor Agent coordinates the workflow lifecycle sequentially.

### 1. Parameter Slot Extraction
- **Operation**: The supervisor runs `getPlannerExtractionPrompt()` against the last few turns of the conversation history.
- **Target Slots**: It parses values for:
  - `destination`, `origin`, `start_date`, `end_date`, `travelers`, `budget_inr`, `interests`, `max_price_per_night`.
- **Programmatic Overrides**:
  - The supervisor extracts lodging price constraints directly via regex to prevent LLM scaling errors (*e.g., "stay below ₹3000"* sets `max_price_per_night`).
  - Strict validator checks execute immediately:
    - `clampTravelers`: Keeps traveler size between `1` and `10`.
    - `clampBudget`: Limits total budget input between `₹5,000` and `₹5,00,000` to prevent computation loops.
    - `validateTripDates`: Ensures dates are in the future and that `start_date <= end_date`.

### 2. Routing Decision
The supervisor invokes a routing model configured with three specific tools:
1. `validate_trip_inputs`
2. `recommend_destination`
3. `orchestrate_and_generate_trip_plan`

#### 🛡️ Hallucination Override Guards
Before executing the tool chosen by the LLM, the supervisor applies strict logical guards:
- **Guard 1**: If context has no `destination`, the router must run `recommend_destination`.
- **Guard 2**: If critical params (`origin`, `dates`, `travelers`, `budget`) are missing, the router must run `validate_trip_inputs`.
- **Guard 3**: If all critical parameters are fully resolved, it forces the tool to `orchestrate_and_generate_trip_plan`.

---

## 🔄 Step 5: Route Logic Processing

Based on the supervisor route, one of three flows is executed:

### Flow A: Validate Trip Inputs (Missing Info Agent)
- [missingInfoAgent.ts](file:///d:/Presidio%20Capstone%20Project/server/src/agents/missingInfoAgent.ts) analyzes the missing fields.
- It drafts a clarifying question highlighting the needed slots.
- Returns status `NEEDS_INFO` back to the frontend.

### Flow B: RECOMMEND DESTINATION (Destination Agent)
- [destinationRecAgent.ts](file:///d:/Presidio%20Capstone%20Project/server/src/agents/destinationRecAgent.ts) combines the traveler's interests, budget constraints, and long-term memory.
- It recommends a primary target destination and provides context on why it matches their preferences.
- If other critical parameters are still missing, it appends a message prompting the user for the remaining slots.

### Flow C: ORCHESTRATE AND GENERATE TRIP PLAN
If all details are available, the supervisor executes the full planning swarm:

```
                  +--------------------------------+
                  |  PlannerAgent (Supervisor)     |
                  +---------------+----------------+
                                  |
                   Trigger swarm  |
                                  v
                  +---------------+----------------+
                  | coordinatorAgent.ts            |
                  | - runParallelAgents()          |
                  +-------+---------------+--------+
                          |               |
             +------------+               +------------+
             |                                         |
             v                                         v
   +---------+---------------+               +---------+---------------+
   | Parallel Fetch stage    |               | Parallel Fetch stage    |
   | - weatherTool           |       ...     | - accommodationTool     |
   | - transportTool         |               | - activityTool          |
   +---------+---------------+               +---------+---------------+
             |                                         |
             +------------+               +------------+
                          |               |
                          v               v
                  +-------+---------------+--------+
                  | budgetAgent.ts                 |
                  | - runBudgetAgent (early check) |
                  +---------------+----------------+
                                  |
                                  v
                  +---------------+----------------+
                  | itineraryAgent.ts              |
                  | - runItineraryAgent (batching) |
                  +---------------+----------------+
                                  |
                                  v
                  +---------------+----------------+
                  | localTransitAgent.ts           |
                  | - runLocalTransitAgent         |
                  +---------------+----------------+
                                  |
                                  v
                  +---------------+----------------+
                  | coordinatorAgent.ts            |
                  | - synthesizeTripPlan()         |
                  +--------------------------------+
```

#### 1. Stage 1: Parallel retrieval ([coordinatorAgent.ts](file:///d:/Presidio%20Capstone%20Project/server/src/agents/coordinatorAgent.ts))
- Orchestrates and fires parallel async tool promises:
  - **Weather Agent** ([weatherAgent.ts](file:///d:/Presidio%20Capstone%20Project/server/src/agents/weatherAgent.ts)): Queries the weather MCP server for forecasts and translates them into an activity reasoning summary via an LLM.
  - **Transport Agent** ([transportAgent.ts](file:///d:/Presidio%20Capstone%20Project/server/src/agents/transportAgent.ts)): Queries the transit MCP server for inter-city travel routes (flights, buses, trains) with prices, departure/arrival hubs, and travel times.
  - **Accommodation Agent** ([accommodationAgent.ts](file:///d:/Presidio%20Capstone%20Project/server/src/agents/accommodationAgent.ts)):
    - Searches real hotel pricing via the Hotelbeds API.
    - Resolves additional local properties via Geoapify Places.
    - Synthesizes fallback/personalized recommendations via an LLM.
    - Merges and deduplicates these entries.
    - Classifies hotels into **Budget**, **Mid-Range**, and **Luxury** tiers based on price thresholds.
  - **Activity Agent** ([activityAgent.ts](file:///d:/Presidio%20Capstone%20Project/server/src/agents/activityAgent.ts)): Fetches sights, architectural spots, and restaurants using Geoapify category filters, ranking them by popularity.
- Programmatic overrides correct any LLM naming errors (e.g. preventing the LLM from swapping origin and destination cities in tool payloads).

#### 2. Stage 2: Budget Evaluation Gate ([budgetAgent.ts](file:///d:/Presidio%20Capstone%20Project/server/src/agents/budgetAgent.ts))
- Calculates estimates for transport, lodging, food, and activities.
- Calculates an emergency fund (10% of subtotal) and checks feasibility against the user's budget.
- **Fail-Safe**: If costs exceed the budget, it halts execution early, generates a list of budget alternatives (*e.g., cheaper hotel tiers, fewer days, reduced traveler count*), and returns a recommendation prompt containing these suggestions.

#### 3. Stage 3: Itinerary Construction ([itineraryAgent.ts](file:///d:/Presidio%20Capstone%20Project/server/src/agents/itineraryAgent.ts))
- Generates a detailed, day-by-day JSON schedule.
- **Batching Strategy**: Iterates through the trip dates in batches of maximum 5 days per LLM call. This prevents prompt length truncation, keeping the returned JSON structure valid.
- Integrates matched weather advisory slots, meal times, and sightseeing recommendations.

#### 4. Stage 4: Local Commute Calculations ([localTransitAgent.ts](file:///d:/Presidio%20Capstone%20Project/server/src/agents/localTransitAgent.ts))
- Automatically processes the generated JSON itinerary items.
- Identifies all non-hotel sightseeing destination addresses.
- Queries Geoapify Directions in parallel to calculate distances from the hotel to each attraction.
- Maps commute paths from the arrival airport or railway station to the hotel.
- Annotates schedule items with transportation guidelines (walk times, auto/cab duration, estimated round-trip costs).
- **Anti-Inflation Safeguard**: Caps the daily local commute budget estimate to a maximum of ₹500 per traveler per day. This prevents geocoding lookup failures from artificially inflating the total estimated cost.
- Re-calculates and finalizes the trip budget.

#### 5. Stage 5: Summary Synthesis
- Invokes `synthesizeTripPlan` on the coordinator.
- Condenses all data records into a structured summary to prevent token limit errors (429 rate limits), then builds the formatted Markdown plan.
- The `TripContext` status is updated to `PLANNED`.

---

## 🌐 Step 6: Database Persistence & Frontend Delivery

### 1. MongoDB Save
- `plannerService.ts` trims the conversation history list down to the latest 50 entries to optimize database storage.
- It updates the database record for the trip, saving the updated inputs, budget estimates, local transport details, and the synthesized Markdown plan.
- If the plan was generated successfully, the user's **Long-Term Memory** is updated with the destination and traveler parameters.

### 2. Client Side Hydration
- The server returns the payload to [ChatPage.tsx](file:///d:/Presidio%20Capstone%20Project/client/src/pages/ChatPage.tsx).
- The state updates, rendering the generated Markdown plan in the UI.
- The dashboard interface renders:
  - **ItineraryTimeline**: Displays the annotated day-by-day timeline, including activities, dining locations, transit modes (auto/cab/walking), and estimated travel costs.
  - **InspectorTab**: Coordinates tabs displaying Weather Forecast cards, Hotel options categorized by price tier (Budget, Mid-Range, Luxury), and Transport options with inter-city distance details. It also renders the budget breakdown chart.

---

## 🛠️ Step 7: Interactive Customizations & Human-in-the-Loop (HITL)

If the traveler wants to modify the generated plan, they can do so using the interactive controls in the UI:

```
                            +---------------|
                            |  Traveler UI   |
                            +-------+-------+
                                    |
            +-----------------------+-----------------------+
   Choose   |                                      Choose   |   Modify & Click
   Hotel    |                                    Transport  |  "Reject & Replan"
            v                                               v       v
+-----------+------------+                      +-----------+------------+
| POST /select-hotel     |                      | POST /select-transport |
| - Updates hotel context|                      | - Updates transport opt|
| - Programmatically     |                      | - Preserves transit    |
|   re-runs Budget &     |                      | - Calibrates and checks|
|   Transit calculations |                      |   budget feasibility   |
+-----------+------------+                      +-----------+------------+
            |                                               |
            +-----------------------+-----------------------+
                                    |
                                    v
                        +-----------+------------+
                        |  Save context state    |
                        |  & regenerate markdown |
                        |  trip plan presentation|
                        +------------------------+
```

### 1. Hotel Category Allocation
- **Action**: The user selects a specific hotel or chooses the "Self Arranged" option.
- **Route**: `POST /api/trips/:tripId/select-hotel`.
- **Handling**:
  - The controller locates the selected hotel and moves it to the front of the options list.
  - It programmatically updates all hotel name occurrences in the itinerary and the Markdown plan.
  - It runs the Budget and Local Transit agents again to update commute costs and check budget feasibility.
  - If the selection exceeds the budget, the trip is changed back to `DRAFT` status, and suggestions for cheaper options are shown. Otherwise, the status is set to `PLANNED`.
  - Re-saves the trip context to MongoDB and returns it to the client.

### 2. Transport Option Selection
- **Action**: The traveler chooses a specific transport recommendation or selects "Self Arranged".
- **Route**: `POST /api/trips/:tripId/select-transport`.
- **Handling**:
  - Saves the selected transport option to the backend context.
  - Recalculates the budget while maintaining the existing local transit calculations (avoiding redundant API queries).
  - Evaluates budget feasibility against the limit, updates the trip status, and saves the updated context.

### 3. Rejection & Replanning Requests
- **Action**: The traveler enters feedback (e.g. *"increase budget to ₹60,000 and select a mid-range hotel"*) and clicks **Reject & Replan**.
- **Route**: `POST /api/trips/:tripId/reject`.
- **Orchestration**:
  - `rejectTrip` controller loads the context and delegates to **Replanning Agent** ([replanningAgent.ts](file:///d:/Presidio%20Capstone%20Project/server/src/agents/replanningAgent.ts)).
  - The agent compares the feedback against the current plan parameters and selects the appropriate replanning tool:
    - `replan_accommodation`: Resets stay selection and budget, forcing a fresh lookup.
    - `replan_dates`: Resets weather, transport, accommodation, itinerary, and transit data.
    - `replan_activities`: Resets activities, itinerary, and transit.
    - `replan_budget`: Resets the budget, triggering a recalculation.
    - `replan_itinerary`: Resets the generated day-by-day JSON schedule.
    - `replan_full_trip`: Clears all generated context data.
  - **Efficiency**: The replanning agent only clears data that is affected by the requested changes, keeping the rest of the generated context intact. This speeds up the update process and reduces external API calls.
  - After updating the context, the system calls `planTrip()` to generate the revised plan options.

---

## 🔒 Step 8: Confirmed Booking Approval

Once the traveler is satisfied with the accommodation, transport, and itinerary details:

- **Action**: The user clicks **Confirm Booking** on the confirmation interface.
- **Route**: `POST /api/trips/:tripId/approve`.
- **Processing**:
  - The backend verifies that the trip status is `PLANNED` and that the overall cost is within the budget limits.
  - If the itinerary is missing, it runs the Itinerary and Local Transit agents to generate the required details before finalizing.
  - Calls **Booking Agent** ([bookingAgent.ts](file:///d:/Presidio%20Capstone%20Project/server/src/agents/bookingAgent.ts)).
  - The booking agent connects to the Google Calendar API ([calendarMCP.ts](file:///d:/Presidio%20Capstone%20Project/server/src/mcp-servers/calendarMCP.ts)) to automatically add the trip dates to the user's calendar.
  - Generates confirmation reference numbers for the selected hotel and transport:
    - *Hotel Confirmation*: `HB-HTL-[HOTEL_PREFIX]-[RANDOM_DIGITS]`
    - *Transport Confirmation*: `PNR-[MODE]-[OPERATOR]-[RANDOM_DIGITS]`
  - Updates the trip status to `CONFIRMED` in the database, locking the plan from further modifications.
  - Returns the confirmation details and updated trip model back to the client.
