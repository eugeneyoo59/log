# 2026-07-19: Annotating proximity files until I can explain every line cold.
## app/matches/_layout.tsx
* The point of this file is to organize how the .tsx files in the matches folder, under the (tabs) grouping, is going to be organized.
* In this case we are working in a stack, as we will shift around screens that need to be peeled off
* We import the stack navigation type from the expo-router package. 

`MatchesLayout()`
* This returns a stack navigator with additional config on the index file, but no config on matchId. I believe that it is okay to remove the `<Stack.Screen name="[matchId]" />` line because the router already knows it is in this folder based on the file tree. 
* The options in `<Stack.Screen>` are options that define whether the header is shown in index. This can be overridden within index itself.

## app/matches/index.tsx

`type FetchedMatch` 
* The file needs to fetch a bunch of matches, including their last line of message, so they can determine what to show and what to bold as a sign of the message not being read yet. 
* This is a type, meaning its just a device that checks for object adherence to the rules of this structure.

`formatTimestamp`
* This function essential formats the date for the last message.
* This function takes an iso (an iso date string) thats typed to string, then returns string. 
* It then forms a new Date object with that iso, then another date object with the current moment. 
* isToday is a boolean if the year, month, date of now and date is equal. If true, we extract the hours and minutes of the day and return that. Else, we return the day the message was sent in mo/day format. 
* padStart(2, '0') turns strings into min 2 length strings by adding 0s to the start. "9" > "09"

` MatchesListScreen()`
* First we set up a bunch of state variables. **Question: when do these refresh? I need to study the mount/unmount dynamics, and when they're triggered and by which file.**
* `loadMatches` loads matches through a `useCallBack` function. useCallBack basically memoizes the function in its parameter across different re-renders. 
* Inside this useCallBack is the actual callback, which first fetches the authorized user from supabase (await call). If there is no user **(what exactly does `!user` mean -- errors, anything except the user type?)** it sets the current loading state as false, and returns from `loadMatches`.
* We then fetch the supabase rows from the matches table where either the user1_id or user2_id matches the user.id from the user we fetched earlier. If there is a matchError, we log an error, set loading as false again, then return. 
* We set match IDs to be equal to the ids extracted from the matchRows object returned by supabase, by making an array of just ids. 
* We then make a constant -- `rows` which is a list of fetched matches. We create this constant by going through each matchin the matchRows, and running them through an arrow function via .map(). We identify the otherUserId, then simultaneously request the other users' profile information, all messages in that match, and the other user's photo.
* We then return a list of FetchedMatch objects that contains information from all matches. 
* Afterwards, we sort `rows` based on the most recent messages, and set the Matches state as rows, set loading as false. **what is !.lastMessageAt? need to read some of the details here.**
