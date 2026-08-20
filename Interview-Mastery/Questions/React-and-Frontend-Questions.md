# React and Frontend Questions

## Questions

1. What is React?
2. What are components?
3. What is JSX?
4. What is state?
5. What are props?
6. Difference between state and props.
7. What are hooks?
8. What is `useState`?
9. What is `useEffect`?
10. What is `useMemo`?
11. What is `useCallback`?
12. What is controlled component?
13. What is uncontrolled component?
14. What is conditional rendering?
15. What is list rendering?
16. Why is key needed in list rendering?
17. What is React Router?
18. What is RBAC in frontend?
19. How do you protect routes in React?
20. How do you call APIs from React?
21. How do you handle loading state?
22. How do you handle errors in UI?
23. How do you optimize React performance?
24. What is lazy loading?
25. What is code splitting?
26. How do you build dashboards in React?
27. What are customizable dashboard grids?
28. What is Three.js?
29. How do you use Three.js for digital twin visualization?
30. How do you show live sensor hotspots in UI?

---

## Answers

1. What is React?
   - **Answer:**
      - React is a JavaScript library for building UIs with a component-based architecture and declarative rendering
      - React can power real-time monitoring dashboards with reusable components for sensor cards, alert panels, and time-series charts
      - React's virtual DOM improves dashboard performance by minimizing actual DOM updates — critical for rendering frequently updating sensor data without UI jank
2. What are components?
   - **Answer:**
      - Components are reusable, self-contained UI pieces that manage their own state and rendering
      - Typical components in a dashboard include `SensorCard` (temperature/humidity), `AlertBanner` (excursion warnings), and `TimeSeriesChart` (historical data)
      - A component hierarchy follows a clear structure: a `Dashboard` composed of `Sidebar`, `Header`, `SensorGrid` with individual `SensorCard` components receiving data via props
3. What is JSX?
   - **Answer:**
      - JSX is a syntax extension for JavaScript that looks like HTML but compiles to React elements
      - For example, `<SensorCard sensor={sensor} onSelect={handleSelect} />` is cleaner than nested `React.createElement` calls
      - JSX expressions use `{}` to embed any valid JavaScript, and Babel transpiles JSX into `React.createElement` calls during the build step
4. What is state?
   - **Answer:**
      - State is data that changes over time within a component, triggering re-renders when updated
      - For example, a `sensorData` state holding the latest readings from an API, and updating it automatically re-renders the sensor cards
      - Component state (`useState`) handles local UI concerns, while global state (Context API or a state library) shares data across distant components. State lifting places shared state in the closest common ancestor so siblings can access it via props
5. What are props?
   - **Answer:**
      - Props are read-only inputs passed from parent to child components, like function arguments
      - For example, a `SensorGrid` passing sensor objects as props to each `SensorCard`: `<SensorCard name={s.name} value={s.temperature} />`
      - Prop drilling occurs when props must pass through intermediate components that don't use them — solved by Context API. TypeScript interfaces enforce prop types at compile time, and default props (or destructuring defaults) provide fallbacks for optional values
6. Difference between state and props.
   - **Answer:**
      - State is mutable data managed within a component; props are immutable data passed from a parent
      - For example, sensor data may arrive as props from the parent, while UI state like "which panel is expanded" is local component state
      - React enforces unidirectional data flow: state set in a parent component flows down as props to children, and children communicate back up by invoking callback props (e.g., `onSelect`, `onClose`)
7. What are hooks?
   - **Answer:**
      - Hooks let functional components use state and lifecycle features
      - Common hooks include `useState` for UI state, `useEffect` for API calls, `useMemo` for expensive computations, and `useCallback` for stable callback references
      - Rules of hooks: only call hooks at the top level of a function (never inside loops, conditions, or nested functions) and only from React function components or custom hooks — these must be followed strictly to avoid unpredictable behavior
8. What is `useState`?
   - **Answer:**
      - `useState` returns a state variable and a setter that triggers re-renders
      - For example, `const [selectedSensor, setSelectedSensor] = useState(null)` can manage which sensor detail panel is open
      - Initial state can be a value or a lazy initializer function for expensive computations. State updates are batched and asynchronous, so use the functional form `setCount(prev => prev + 1)` when the new state depends on the previous value
9. What is `useEffect`?
   - **Answer:**
      - `useEffect` runs side effects after rendering - API calls, subscriptions, DOM updates
      - For example, it can fetch sensor data on mount and set up polling for live updates
      - The cleanup function (returned from the effect) runs before unmount or before the next effect execution — used to clear `setInterval` or unsubscribe from WebSocket. An empty dependency array `[]` runs the effect once on mount; omitting the array runs it after every render; listing dependencies runs it only when those values change. Always include all referenced state/props in the dependency array to avoid stale closures and infinite loops
10. What is `useMemo`?
    - **Answer:**
       - `useMemo` memoizes an expensive computation result, recalculating only when dependencies change
       - For example, it can compute aggregated statistics from partner data, avoiding recalculation on every render
       - `useMemo` memoizes a **value**, while `useCallback` memoizes a **function**. Use `useMemo` for costly operations (sorting, filtering large arrays, complex calculations) but avoid it for cheap computations where memoization overhead exceeds the savings
11. What is `useCallback`?
    - **Answer:**
       - `useCallback` memoizes a function reference, preventing child re-renders when the reference changes unnecessarily
       - For example, wrapping a `handleSensorSelect` callback in `useCallback` prevents child `SensorCard` components from re-rendering for unrelated parent state changes
       - `useCallback(fn, deps)` is essentially `useMemo(() => fn, deps)` under the hood. It is most useful when passing callbacks to child components wrapped in `React.memo` — without memoization, a new function reference on every parent render would defeat `React.memo` and cause unnecessary child re-renders
12. What is controlled component?
    - **Answer:**
       - A controlled component has its value controlled by React state: `<input value={filterText} onChange={(e) => setFilterText(e.target.value)} />`
       - React state is the single source of truth
       - In contrast, uncontrolled components let the DOM manage their own state, accessed via `useRef`. Controlled components are preferred for form validation, dynamic UIs, and testability since the component state can be asserted directly in tests
13. What is uncontrolled component?
    - **Answer:**
       - An uncontrolled component manages its own state via the DOM, accessed through refs
       - Uncontrolled components can be simpler for forms that only need values on submit
       - Access the current value with `useRef`: `const inputRef = useRef(); <input ref={inputRef} />`. They are less testable since form state lives in the DOM rather than React state, making it harder to assert against in unit tests
14. What is conditional rendering?
    - **Answer:**
       - Conditional rendering shows different UI based on conditions using ternaries, `&&`, or if-else in JSX
       - For example, `{sensor.status === 'alert' && <AlertIcon />}` shows alert icons only for abnormal readings
       - Ternary `condition ? <A /> : <B />` handles if-else cases. The `&&` operator is a shorthand for show/hide when the else case renders nothing. For complex multi-branch logic, extract to a helper function that returns JSX or use an early return pattern before the JSX return
15. What is list rendering?
    - **Answer:**
       - List rendering uses `Array.map()` to transform data arrays into React elements
       - For example: `{partners.map(p => <PartnerRow key={p.id} partner={p} />)}`
       - Always handle empty lists by showing a "no data" message (e.g., `{items.length === 0 && <EmptyState />}`). Display loading states during data fetches, and for large lists use pagination or virtualization (react-window) to render only visible items efficiently
16. Why is key needed in list rendering?
    - **Answer:**
       - Keys help React identify which items changed, were added, or removed, enabling efficient DOM updates
       - Always use unique IDs as keys — never array indices, which cause incorrect re-rendering when list order changes
       - Keys must be stable, unique, and predictable across renders — never generate them during render (e.g., `Date.now()` or `Math.random()`). Using array index as a key is acceptable only for static, non-reordered, non-filtered lists where items never change position
17. What is React Router?
    - **Answer:**
       - React Router enables client-side routing for SPAs without full page reloads
       - Routes like `/dashboard`, `/alerts`, `/reports`, and `/settings` can be set up with lazy loading
       - `BrowserRouter` uses the History API for clean URLs (preferred for most apps), while `HashRouter` uses URL hash fragments (better for legacy servers). Protected routes wrap route components with auth-check logic. Route parameters (`/sensors/:id`) pass dynamic segments accessible via `useParams`
18. What is RBAC in frontend?
    - **Answer:**
       - RBAC restricts UI elements based on user role
       - For example, admins may see "Settings" and "Delete" buttons, while viewers see read-only dashboards
       - Role info typically comes from JWT claims
       - Frontend RBAC is a UX convenience, not a security measure — all sensitive operations must be enforced on the backend regardless of what the UI hides. The frontend simply reduces confusion by not showing actions the user cannot perform
19. How do you protect routes in React?
    - **Answer:**
       - A `ProtectedRoute` wrapper checks auth state from the JWT in localStorage
       - If authenticated, the route renders; otherwise, it redirects to login using React Router's `Navigate`
       - Role-based checks can be added for admin-only routes
       - Token expiry can be handled by decoding the JWT `exp` claim and redirecting to login with a "session expired" message when the token is stale, prompting re-authentication
20. How do you call APIs from React?
    - **Answer:**
       - The Fetch API with async/await in `useEffect` is a common pattern, along with a centralized API utility handling base URL, JWT headers, and error handling
       - For example, `useEffect` can poll every 30 seconds and update state
       - Axios is an alternative that provides built-in request/response interceptors for automatic token refresh, cleaner error handling via `response.data`, and native `AbortController`-based request cancellation to prevent state updates on unmounted components
21. How do you handle loading state?
    - **Answer:**
       - Loading can be managed with a boolean: `const [loading, setLoading] = useState(true)`, set to false after the API response
       - The UI shows a spinner while loading, replaced by data or an error message
       - Skeleton screens provide better perceived performance than spinners by showing content-shaped placeholders. For component-level loading (not page-level), each widget can manage its own loading state independently so one slow API call doesn't block the entire dashboard
22. How do you handle errors in UI?
    - **Answer:**
       - Error states can be displayed with an `ErrorBanner` showing a descriptive message and optional retry button
       - API errors are caught in the fetch, and the error message is extracted from the response body
       - Network errors can show "connection lost" with auto-retry
       - React Error Boundaries (class components with `componentDidCatch`) catch rendering errors in child component trees and display a fallback UI instead of crashing the entire dashboard — each major dashboard section can be wrapped in its own boundary so a failed chart doesn't take down the whole page
23. How do you optimize React performance?
    - **Answer:**
       - Common optimizations include `React.memo` for pure components, `useMemo`/`useCallback` for expensive work, code splitting with lazy loading, and keeping state as local as possible to minimize re-renders
       - React DevTools Profiler identifies which components re-render and how long they take, guiding targeted optimizations. `react-window` virtualizes large lists by rendering only visible rows. Debouncing rapid state updates (e.g., search input, WebSocket data) prevents excessive re-renders by batching changes into a single update
24. What is lazy loading?
    - **Answer:**
       - Lazy loading defers component loading until needed, reducing initial bundle size
       - For example, a `ReportsPage` can be lazy-loaded: `const ReportsPage = React.lazy(() => import('./ReportsPage'))`, loading only when the user navigates to reports
       - Wrap lazy components in `<Suspense fallback={<Spinner />}>` to show a loading indicator while the chunk loads. Webpack handles chunking automatically with dynamic `import()`, creating separate JS files that are fetched on demand by the browser
25. What is code splitting?
    - **Answer:**
       - Code splitting breaks the JS bundle into smaller chunks loaded on demand, improving initial page load
       - Route-based splitting places each route's component in a separate chunk, loaded only when visited
       - This can reduce initial load from ~1MB to ~200KB
       - Webpack dynamic imports (`import('./Module')`) create split points in the bundle. `source-map-explorer` visualizes chunk sizes to identify optimization opportunities. Vendor libraries (React, lodash) can be split into a separate chunk cached across deploys since they change infrequently
26. How do you build dashboards in React?
    - **Answer:**
       - Dashboards can be built with a modular component architecture: a `DashboardLayout` with configurable grid areas, reusable widgets (`SensorWidget`, `AlertWidget`, `ChartWidget`), and a data layer polling an API
       - Each widget is independent
       - react-grid-layout provides drag-and-drop widget positioning and resizing. Each widget fetches its own data independently so a slow or failing API call in one widget doesn't block others — loading and error states are scoped per-widget, not per-page
27. What are customizable dashboard grids?
    - **Answer:**
       - Customizable grids let users rearrange widgets via drag-and-drop, resize them, and save layout preferences
       - react-grid-layout can be used, storing the layout JSON in localStorage or sending it to the backend for persistence
       - Responsive breakpoints (`lg`, `md`, `sm`, `xs`) define different column counts and widget positions for various screen sizes. Layout preferences can be persisted per-user so returning to the dashboard restores each user's preferred widget arrangement across sessions
28. What is Three.js?
    - **Answer:**
       - Three.js is a 3D JavaScript library using WebGL for rendering 3D scenes in browsers
       - It can be used to create a 3D digital twin of a facility with sensor locations and real-time temperature hotspots
       - Core concepts: `Scene` holds all 3D objects, `Camera` (PerspectiveCamera or OrthographicCamera) defines the viewpoint, and `Renderer` draws the scene to a canvas element. Sensor coordinates can be mapped from real-world positions to 3D space coordinates within the model
29. How do you use Three.js for digital twin visualization?
    - **Answer:**
       - A 3D facility model can be built with floor plans, shelving units, and sensor spheres at physical locations
       - Sensor readings can be color-mapped (green=normal, yellow=warning, red=alarm), and clicking a sensor sphere shows its real-time data panel
       - Geometry (floor plans, shelves) can be loaded from JSON model files exported from a CAD tool. Real-time color updates use `requestAnimationFrame` to refresh sensor sphere materials on each frame. Rendering hundreds of sensor spheres requires instanced meshes and frustum culling to maintain 60fps performance
30. How do you show live sensor hotspots in UI?
    - **Answer:**
       - Live hotspots are color-coded overlays on the 3D model
       - Each sensor sphere color updates based on latest temperature (blue=cold, red=hot), and a heatmap effect interpolates colors between sensor positions
       - A `useEffect` with `setInterval` polls the API for new readings, then updates Three.js `MeshStandardMaterial` color properties on each sensor sphere. The heatmap effect uses bilinear interpolation between adjacent sensor positions to smoothly blend colors, creating a continuous temperature gradient across the floor
