# Proposal

The submitted version is your Canvas answer for m8a1. This copy lives in the
repository so the plan and the code sit next to each other.

Paste or rewrite the proposal here, and **keep it updated** as things change. A
proposal that still describes a feature you cut in October is worse than no
proposal.

## The parts most likely to drift

- **Core features.** Move anything you cut to stretch goals rather than deleting
  it. The record of what you cut, and why, is worth marks.
- **Where each piece is hosted.** Client, API, database, and the free tier's
  catch for each. If you change host, note the date and the reason.
- **The date demo mode goes off.** If that date has passed and it is still on,
  that is the most important line in this file.
- **Risks.** Which have shrunk, which grew, which turned out to be nothing.

# Final project - proposal

## The idea

Barkada League is a web application that allows a small group of friends to record their 1v1 match results and automatically track their standings in a league. It provides a leaderboard, match history, and individual player statistics based on the results of recorded matches.

## Why

The idea came from how groups of friends often compete against each other in games but don't have a proper way of keeping track of who's winning overall. Usually, results are remembered, discussed in group chats, or recorded manually, which can become confusing over time.

I want to create a simple system where we can record match results in one place and let the application calculate the standings for us. Instead of manually updating wins, losses, and rankings, everything should update automatically whenever a match is added, edited, or deleted.

Another reason for choosing this project is to apply what I've learned about React, Express, REST APIs, and PostgreSQL by connecting them into one working full-stack application.

## Scope

**What's included in the first version:**

- A leaderboard that automatically ranks players based on their match results.
- A form for recording 1v1 matches, including the players, scores, date, and season.
- Match history where users can view, edit, and delete recorded matches.
- Individual player profiles showing wins, losses, win percentage, and current streak.
- A PostgreSQL database that stores players, seasons, and match records.
- An Express REST API that handles requests, validates match data, and calculates player statistics.
- A responsive React interface that works on desktop and mobile devices.

**What's deliberately left out:**

- User registration and login, since the application is intended for a small, private group of friends.
- Tournament brackets and team-based matches, since the focus is only on 1v1 competitions.
- Real-time multiplayer features or live match tracking.
- Automatic player and season management through the interface. These can be configured directly in the database for the first version.
- Advanced analytics, notifications, and other features that aren't necessary for recording matches and tracking standings.

## Milestones

- **Milestone 1: Planning and interface design.** Define the application's features, create wireframes, establish a design system, and build the initial React pages using sample data.
- **Milestone 2: Database and backend development.** Design the PostgreSQL tables for players, seasons, and matches. Build the Express API endpoints for retrieving and managing match records.
- **Milestone 3: Leaderboard and statistics.** Implement the calculations for wins, losses, win percentages, rankings, and winning or losing streaks using actual match data.
- **Milestone 4: Full-stack integration.** Connect the React frontend to the Express API, replace mock data with real database records, and test the complete process of adding, editing, and deleting matches.
- **Milestone 5: Testing and deployment.** Test the application on desktop and mobile, fix bugs, deploy the frontend and backend, and prepare the documentation and final presentation.

## Open questions

- How can I make sure the leaderboard and player statistics stay accurate whenever a match is edited or deleted?
- What is the best way to structure the API so the frontend can retrieve match history and calculated statistics without repeating the same logic?
- How should I handle invalid match results, such as tied scores, negative scores, or selecting the same player twice?
- How can I keep the application simple enough to finish within the given timeframe while still making it useful for a small group of friends?
- How can I deploy the frontend, backend, and database so they can communicate reliably outside my local development environment?
