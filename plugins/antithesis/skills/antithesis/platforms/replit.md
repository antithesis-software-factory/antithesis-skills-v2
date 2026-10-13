# Replit

- Antithesis credentials come from the App's Secrets. If `ANTITHESIS_TENANT` or `ANTITHESIS_API_KEY` is missing, tell the user to add it there, then stop. Never ask the user to paste the API key into chat.
- Replit chats can't read secrets. If you are running in a chat rather than an App, tell the user to run the request from an App that has the secrets, then stop.
- If a module tells you to use a subagent and you can't start one, do that work yourself, one item at a time.
