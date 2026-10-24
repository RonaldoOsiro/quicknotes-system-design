# QuickNotes System Design & Frontend Client

## Project Description
QuickNotes is a note-taking application project featuring a frontend API client and system design specifications for scaling to 1 million active users.

## How to Run the API Client
1. Clone the repository and navigate to the project directory.
2. Open `index.html` in a modern web browser or use a local development server.
3. Click **Load Notes** to fetch initial mock notes, create new notes via the form, and delete notes instantly.

## System Design Documentation Links
* [API Design Document](docs/api-design.md)
* [Data Model Document](docs/data-model.md)
* [Architecture Document](docs/architecture.md)

## What I Learned
1. **Safer DOM Manipulation**: Learned to use `textContent` when rendering user-generated text so it is not interpreted as HTML.
2. **Asynchronous State & Error Handling**: Gained mastery over `async/await`, `try/catch/finally`, disabling buttons during requests, and clear UI feedback states.
3. **System Design**: Learned how to translate application scale (1 million users) into database indexing, caching strategies, and load estimates.

The API client uses JSONPlaceholder as a mock service. Create and delete requests are simulated and are not persisted by the service.