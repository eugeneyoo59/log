# 2026-07-19: Cont'd - Annotating proximity files until I can explain every line cold.

## app/matches/index.tsx

Continuing with ` MatchesListScreen()`

### `loadMatches()` Cont'd

- `matchRows ?? []` is a guard against the entirety of matchRows being a null object -- which is the case when its null. If will return [] if matchRows is null.
- to select the most recent message only, and the first picture, we first order the returned data ascending/descending, then slice the top with limit(1)`.maybeSingle()` unwraps length-1 arrays.
- Supabase queries return a response object, with an error attrubute typed to null or PostgrestError. Hence why we can do `profileResult.error`.
- Since data and error being null are mutually exclusive, error being non-null (e.g. `if (matchError)`) is sufficient as a signal that the operation failed.
- `profile?.~~~~~` is a separate guard that returns undefined if profile is null or undefined, and doesnt try to access the attribute after profile (because we'd be trying to access an attribute of a null object).
- Promise.all returns a promise for an array of the resolved value of the promises, allowing each component to operate in parallel to each other. In the rows case, its an array of promises as each object in it is a row in matchRows that's being funneled through an async function that relies on a supabase call and returns a FetchedMatch-type object. It just gets turned into ONE promise by Promise.all and then resolves to an array of FetchedMatches.
- Then we sort. If match A has no last message, we say A goes after B. If B has no last message, and A does have a last message, we say A goes before B. If both have a last message, if a's time is closer to the present (more recent) we make sure a's match goes first.
- Then we set the Matches state as rows, then set loading as false.
- User flow in this screen is basically: User moves into matches screen --> screen initially mounts, initializing empty state and rendering JSX (in this case, loading spinner) --> useFocusEffect() calls loadMatches() to update state, which triggers a re-render --> latest matches information is shown on JSX.
- We use `useCallback()` to wrap this matches-loading function to memoize it, making it the same memory object every render. This ensures that re-renders won't trigger useEffect() realtime subscription (which checks reference equality) to supabase to re-run, wasting time/computation.

### Section on `useFocusEffect()`

- Note, useFocusEffect() runs the callback at focus, then stores the return value (called cleanup, expecting a function), then runs the cleanup function at blur.
- That's why we need to wrap the stable loadMatches() function in an arrow function. Because its an async function that returns a promise, and useFocusEffect() checks if the cleanup != null before calling cleanup(). Currently the empty arrow function returns undefined which fails the if statement and doesn't lead to any calls. If we didn't wrap it we'd have a promise returned. It passes the check, and react would try to 'call' a promise, treating it like a function, which is going to cause a crash.
- We also use the useCallback() function for `useFocusEffect(useCallback(() => { loadMatches(); }, [loadMatches]));` because useFocusEffect() treats the callback you pass into the function as a dependency. When the component is re-rendered, everything is re-computed, but since we used useCallback() without any dependencies of its own it will always return the original arrow function -- same object, no re-triggering of useFocusEffect at re-render, and only triggers on focus. This avoids a situation where 1. State changes 2. Usefocus triggers 3. Loadmatches triggers 4. State changes 5. Usefocus triggers...

### Section on `useEffect()`

- useEffect() runs on both mount and re-renders without a dependency array, only on mount if empty dependency array, and specific changes on dependency array elements if it has a nonempty dependency array.
- `const ids = matchIdsKey ? matchIdsKey.split(",") : [];` We create a string array of match IDs. JS will coerce matchIdsKey into a boolean, where non empty is truthy, empty is falsy. So if it exists, we split it by commas into an array of Strings.
- We then set channel to an instance of the supabase realtime channel object and name it matches-list-messages.
- We then configure it: For each of those IDs, we will 1. listen for postgres changes, which are row-level changes in the DB, specifically insert events in the public schema in the messages table, filtering for match_id equalling ID. These turn on n different listening rules/triggers, all of which will cause `loadMatches()` to be called again, because upon receiving a new message we want to update the most recent message's time, content, and message ordering.
- and `channel.subscribe()` formally turns this channel on.
- We have a cleanup function for before a re-render or at unmount where we remove the channel to prevent duplicates.
- Dependencies include changes to loadMatches (stable anyways) and matchIdsKey, suggesting new matches are in, which warrants new listeners being built.
- How are new matches handled? > focus onto this screen → loadMatches() → setMatchIds → re-render → matchIdsKey recomputed → useEffect dependency changed → old channel cleaned up → new channel + listeners created for the updated match list.
