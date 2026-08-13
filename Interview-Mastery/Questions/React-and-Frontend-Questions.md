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
26. How did you build dashboards in React?
27. What are customizable dashboard grids?
28. What is Three.js?
29. How did you use Three.js for digital twin visualization?
30. How do you show live sensor hotspots in UI?

---

## Answers

1. What is React?
   - **Answer:** React is a JavaScript library for building UIs with a component-based architecture and declarative rendering. In our cold-chain project, React powered the real-time monitoring dashboard with reusable components for sensor cards, alert panels, and time-series charts.
   - **If asked more:** I would discuss how React's virtual DOM improved dashboard performance by minimizing actual DOM updates, critical for rendering frequently updating sensor data without UI jank.
2. What are components?
   - **Answer:** Components are reusable, self-contained UI pieces that manage their own state and rendering. In our cold-chain dashboard, we had `SensorCard` (temperature/humidity), `AlertBanner` (excursion warnings), and `TimeSeriesChart` (historical data).
   - **If asked more:** I would explain the component hierarchy: `Dashboard` composed `Sidebar`, `Header`, `SensorGrid` with individual `SensorCard` components receiving data via props.
3. What is JSX?
   - **Answer:** JSX is a syntax extension for JavaScript that looks like HTML but compiles to React elements. In our project, `<SensorCard sensor={sensor} onSelect={handleSelect} />` was cleaner than nested `React.createElement` calls.
   - **If asked more:** I would discuss JSX expressions using `{}` for JavaScript and how Babel transpiles JSX during build.
4. What is state?
   - **Answer:** State is data that changes over time within a component, triggering re-renders when updated. In our cold-chain dashboard, `sensorData` state held the latest readings from the API, and updating it automatically re-rendered the sensor cards.
   - **If asked more:** I would discuss component state (useState) vs global state (Context) and the principle of state lifting to the closest common ancestor.
5. What are props?
   - **Answer:** Props are read-only inputs passed from parent to child components, like function arguments. In our dashboard, `SensorGrid` passed sensor objects as props to each `SensorCard`: `<SensorCard name={s.name} value={s.temperature} />`.
   - **If asked more:** I would discuss prop drilling, TypeScript interfaces for prop validation, and default props for optional values.
6. Difference between state and props.
   - **Answer:** State is mutable data managed within a component; props are immutable data passed from a parent. In our dashboard, sensor data came as props from the parent, while UI state like "which panel is expanded" was local component state.
   - **If asked more:** I would explain unidirectional data flow: state in parents flows down as props, children communicate up via callback props.
7. What are hooks?
   - **Answer:** Hooks let functional components use state and lifecycle features. We used `useState` for UI state, `useEffect` for API calls, `useMemo` for expensive computations, and `useCallback` for stable callback references.
   - **If asked more:** I would mention the rules of hooks: only call at top level, only from React functions. We followed these strictly.
8. What is `useState`?
   - **Answer:** `useState` returns a state variable and a setter that triggers re-renders. In our dashboard, `const [selectedSensor, setSelectedSensor] = useState(null)` managed which sensor detail panel was open.
   - **If asked more:** I would discuss initial state, async updates, and the functional form `setCount(prev => prev + 1)` for state depending on previous value.
9. What is `useEffect`?
   - **Answer:** `useEffect` runs side effects after rendering - API calls, subscriptions, DOM updates. In our cold-chain dashboard, it fetched sensor data on mount and set up polling for live updates.
   - **If asked more:** I would discuss cleanup functions, the dependency array (empty = run once, omitted = run every render), and avoiding infinite loops.
10. What is `useMemo`?
    - **Answer:** `useMemo` memoizes an expensive computation result, recalculating only when dependencies change. In our inventory dashboard, we used it to compute aggregated statistics from partner data, avoiding recalculation on every render.
    - **If asked more:** I would distinguish `useMemo` (memoizes a value) from `useCallback` (memoizes a function).
11. What is `useCallback`?
    - **Answer:** `useCallback` memoizes a function reference, preventing child re-renders when the reference changes unnecessarily. We wrapped `handleSensorSelect` in `useCallback` so child `SensorCard` components did not re-render for unrelated parent state changes.
    - **If asked more:** I would explain that `useCallback(fn, deps)` is essentially `useMemo(() => fn, deps)` and is most useful with optimized child components.
12. What is controlled component?
    - **Answer:** A controlled component has its value controlled by React state: `<input value={filterText} onChange={(e) => setFilterText(e.target.value)} />`. React state is the single source of truth.
    - **If asked more:** I would discuss the difference from uncontrolled components (DOM handles its own state via refs) and why controlled is preferred for validation and dynamic UIs.
13. What is uncontrolled component?
    - **Answer:** An uncontrolled component manages its own state via the DOM, accessed through refs. While we generally used controlled components, uncontrolled can be simpler for forms that only need values on submit.
    - **If asked more:** I would explain that uncontrolled components use `useRef` and are less testable since state is not in React.
14. What is conditional rendering?
    - **Answer:** Conditional rendering shows different UI based on conditions using ternaries, `&&`, or if-else in JSX. In our dashboard, `{sensor.status === 'alert' && <AlertIcon />}` showed alert icons only for abnormal readings.
    - **If asked more:** I would discuss ternary for if-else, `&&` for simple show/hide, and early returns for complex conditions.
15. What is list rendering?
    - **Answer:** List rendering uses `Array.map()` to transform data arrays into React elements. In our dashboard: `{partners.map(p => <PartnerRow key={p.id} partner={p} />)}`.
    - **If asked more:** I would discuss empty list handling ("no data" message), loading states, and rendering paginated lists efficiently.
16. Why is key needed in list rendering?
    - **Answer:** Keys help React identify which items changed, were added, or removed, enabling efficient DOM updates. We always used unique IDs as keys - never array indices, which cause incorrect re-rendering when list order changes.
    - **If asked more:** I would explain that keys must be stable, unique, and predictable. Index as key is acceptable only for static, non-reordered lists.
17. What is React Router?
    - **Answer:** React Router enables client-side routing for SPAs without full page reloads. In our dashboard, it managed routes like `/dashboard`, `/alerts`, `/reports`, and `/settings` with lazy loading.
    - **If asked more:** I would discuss BrowserRouter vs HashRouter, protected routes with auth checks, and route parameters for sensor details.
18. What is RBAC in frontend?
    - **Answer:** RBAC restricts UI elements based on user role. In our dashboard, admins saw "Settings" and "Delete" buttons, while viewers saw read-only dashboards. Role info came from JWT claims.
    - **If asked more:** I would emphasize that frontend RBAC is UX convenience, not security - all sensitive operations must be enforced on the backend.
19. How do you protect routes in React?
    - **Answer:** We created a `ProtectedRoute` wrapper that checked auth state from the JWT in localStorage. If authenticated, the route rendered; otherwise, it redirected to login using React Router's `Navigate`. Role-based checks were added for admin-only routes.
    - **If asked more:** I would discuss handling token expiry by redirecting to login with a session-expired message.
20. How do you call APIs from React?
    - **Answer:** We used the Fetch API with async/await in `useEffect`, with a centralized API utility handling base URL, JWT headers, and error handling. In the cold-chain dashboard, `useEffect` polled every 30 seconds and updated state.
    - **If asked more:** I would discuss Axios as an alternative with interceptors for token refresh and request cancellation for unmounted components.
21. How do you handle loading state?
    - **Answer:** We managed loading with a boolean: `const [loading, setLoading] = useState(true)`, set to false after the API response. The UI showed a spinner while loading, replaced by data or an error message.
    - **If asked more:** I would discuss skeleton screens for better UX and handling loading for individual components vs the whole page.
22. How do you handle errors in UI?
    - **Answer:** We displayed error states with an `ErrorBanner` showing a descriptive message and optional retry button. API errors were caught in the fetch, and the error message was extracted from the response body. Network errors showed "connection lost" with auto-retry.
    - **If asked more:** I would discuss error boundaries to prevent the whole dashboard from crashing due to one failed component.
23. How do you optimize React performance?
    - **Answer:** We optimized with `React.memo` for pure components, `useMemo`/`useCallback` for expensive work, code splitting with lazy loading, and keeping state as local as possible to minimize re-renders.
    - **If asked more:** I would discuss React DevTools Profiler, virtualization with react-window for large lists, and debouncing rapid state updates.
24. What is lazy loading?
    - **Answer:** Lazy loading defers component loading until needed, reducing initial bundle size. In our dashboard, `ReportsPage` was lazy-loaded: `const ReportsPage = React.lazy(() => import('./ReportsPage'))`, loading only when the user navigated to reports.
    - **If asked more:** I would discuss combining lazy loading with Suspense for loading states and Webpack chunking.
25. What is code splitting?
    - **Answer:** Code splitting breaks the JS bundle into smaller chunks loaded on demand, improving initial page load. Our dashboard used route-based splitting: each route's component was in a separate chunk, loaded only when visited. Initial load was ~200KB instead of 1MB.
    - **If asked more:** I would discuss Webpack dynamic imports, analyzing bundle size with source-map-explorer, and splitting vendor libraries for caching.
26. How did you build dashboards in React?
    - **Answer:** We built dashboards with a modular component architecture: `DashboardLayout` with configurable grid areas, reusable widgets (`SensorWidget`, `AlertWidget`, `ChartWidget`), and a data layer polling the Spring Boot API. Each widget was independent.
    - **If asked more:** I would discuss the customizable grid layout using react-grid-layout for drag-and-drop widget positioning and per-widget data fetching to isolate loading/error states.
27. What are customizable dashboard grids?
    - **Answer:** Customizable grids let users rearrange widgets via drag-and-drop, resize them, and save layout preferences. We used react-grid-layout, storing the layout JSON in localStorage or sending it to the backend for persistence.
    - **If asked more:** I would discuss responsive breakpoints for different screen sizes and how layout preferences persisted across sessions.
28. What is Three.js?
    - **Answer:** Three.js is a 3D JavaScript library using WebGL for rendering 3D scenes in browsers. In our cold-chain project, we used it to create a 3D digital twin of the warehouse with sensor locations and real-time temperature hotspots.
    - **If asked more:** I would discuss basic Three.js concepts: scene, camera, renderer, and mapping sensor coordinates to 3D positions.
29. How did you use Three.js for digital twin visualization?
    - **Answer:** We built a 3D warehouse model with floor plans, shelving units, and sensor spheres at physical locations. Sensor readings were color-mapped (green=normal, yellow=warning, red=alarm), and clicking a sensor sphere showed its real-time data panel.
    - **If asked more:** I would discuss loading geometry from JSON, updating colors in real-time using `requestAnimationFrame`, and the performance challenge of rendering hundreds of sensors.
30. How do you show live sensor hotspots in UI?
    - **Answer:** Live hotspots were color-coded overlays on the 3D model. Each sensor sphere color updated based on latest temperature (blue=cold, red=hot), and a heatmap effect interpolated colors between sensor positions.
    - **If asked more:** I would discuss the polling mechanism in `useEffect` with `setInterval`, updating Three.js object material colors, and the interpolation algorithm for the heatmap effect.

