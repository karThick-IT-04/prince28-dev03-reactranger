# ReactRanger - Evaluation Questions

These 10 questions will be used to evaluate ReactRanger's accuracy, documentation grounding, modern React guidance, and ability to identify legacy patterns.

## Assigned questions

1. When should I use `useMemo`?

2. What are React Actions and how do forms use them in React 19?

3. Why does React recommend not fetching data inside `useEffect` when a framework is available?

4. Convert a class component with `componentDidMount` fetching data, a ref to an input, and a controlled form to React 19 with hooks and Actions. Explain the changes.

5. Compare `useState` and `useReducer` for a multi-step form.

6. Why is our client's website slow on the office Wi-Fi?

   Expected behaviour: ReactRanger should acknowledge that it does not know the cause from the information provided. It must not invent a React-related explanation.

## Additional evaluation questions

7. What is the difference between `useEffect` and `useLayoutEffect`, and when should each be used?

8. How does React 19 handle refs in function components, and when is `forwardRef` still relevant?

9. What problems can occur when a React component uses an incorrect dependency array in `useEffect`?

10. How should a developer decide whether state belongs in a component, should be lifted to a parent, or should be managed with a reducer?

## Evaluation principles

- Answers should follow current official React documentation.
- Legacy patterns must be clearly identified.
- Modern alternatives should be explained.
- Important claims should include relevant documentation links.
- When the available information is insufficient, ReactRanger should clearly acknowledge uncertainty instead of inventing an answer.
