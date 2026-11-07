# Rent Now

A full-stack, server-rendered property rental web app (Airbnb-style). Hosts publish listings with photos and a map location, guests browse, search, book and review stays, and AI-generated summaries condense a listing's reviews.

Built with **Node.js, Express 5, MongoDB (Mongoose), EJS, Passport, Cloudinary, Mapbox and Google Gemini**.

---

## Table of Contents

1. [Key Features](#key-features)
2. [Tech Stack](#tech-stack)
3. [Architecture](#architecture)
4. [Folder Structure](#folder-structure)
5. [Data Models](#data-models)
6. [Routes Reference](#routes-reference)
7. [Prerequisites](#prerequisites)
8. [Installation & Setup](#installation--setup)
9. [Environment Variables](#environment-variables)
10. [Commands](#commands)
11. [Seeding Sample Data](#seeding-sample-data)
12. [Usage Walkthrough](#usage-walkthrough)
13. [Deployment](#deployment)
14. [Known Issues & Caveats](#known-issues--caveats)
15. [Contributing Tips](#contributing-tips)

---

## Key Features

**Accounts & sessions**
- Sign up with username, email and password; log in / log out (Passport Local + `passport-local-mongoose`, which hashes and salts passwords).
- Sessions are stored in MongoDB (`connect-mongo`, encrypted with `SECRET`) and last 7 days.
- Visiting a protected page while logged out redirects to `/login`, then returns you to the page you wanted.
- Flash messages for success and error feedback.

**Listings**
- Create, view, edit and delete listings (edit/delete restricted to the listing's owner).
- Image upload to Cloudinary (`png`, `jpg`, `jpeg` allowed) via Multer.
- Location is geocoded with the Mapbox Geocoding API when a listing is created or updated; the listing page shows an interactive Mapbox map with a marker.
- Browse by category (filter bar) and free-text search across title, location and country (case-insensitive).
- A "Display total after taxes" toggle on the index page that shows a `+ 18% GST` label next to each price (display only).
- Prices are shown in INR (₹).

**Bookings**
- Logged-in users can book a listing by choosing start and end dates.
- Total cost is calculated on the server as `nights × price per night` (the client also shows a live preview).
- Owners cannot book their own listing; end date must be after start date.

**Reviews**
- Logged-in users can leave a 1–5 star rating and a comment (star UI from `public/css/rating.css`).
- Review authors can delete their own reviews. Owners cannot review their own listing.
- When a listing has **3 or more** reviews, the listing page calls Google Gemini (`gemini-2.5-flash`) to produce a 2–3 sentence summary, shown as an "AI Summary" box.

**Dashboard**
- `/dashboard` hub with **My Bookings** (bookings you made) and **My Listings** (properties you host).

**Platform**
- Server-side validation with Joi (listings and reviews), centralized error page, `helmet` security headers (Content Security Policy is disabled, see [Known Issues](#known-issues--caveats)), Vercel configuration included.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Runtime | Node.js (`engines.node` is `18.20.4`) |
| Web framework | Express `^5.1.0` |
| Templating | EJS `^3.1.10` with `ejs-mate` layouts |
| Database | MongoDB via Mongoose `^8.15.0` |
| Auth | Passport `^0.7.0`, `passport-local`, `passport-local-mongoose` |
| Sessions | `express-session`, `connect-mongo` (Mongo-backed), `connect-flash` |
| Validation | Joi `^17.13.3` |
| File upload | Multer `^2.0.2` + `multer-storage-cloudinary` + Cloudinary SDK `^1.32.0` |
| Maps / geocoding | `@mapbox/mapbox-sdk` (server), Mapbox GL JS `3.15.0` (browser, via CDN) |
| AI | `@google/genai` `^1.25.0` (Gemini `gemini-2.5-flash`) |
| Security | `helmet` |
| Utilities | `method-override` (PUT/DELETE from forms), `dotenv` |
| Front-end | Bootstrap 5.3.6, Font Awesome 6.7.2, Google Font "Plus Jakarta Sans" (all via CDN), custom CSS |
| Dev tooling | `nodemon` (invoked by `npm run dev`; **not listed in `package.json`**, see [Commands](#commands)) |
| Hosting config | Vercel (`vercel.json`, `@vercel/node`) |

---

## Architecture

The app is a classic **MVC monolith**: Express renders EJS views on the server and there is no separate front-end build step. Its routes follow **RESTful conventions** (resource URLs such as `/listings` and `/listings/:id`, with GET/POST/PUT/DELETE; PUT and DELETE are sent from HTML forms via `method-override`), but every handler responds with a rendered HTML page or a redirect, not JSON. `express.json()` is enabled in `app.js`, yet no endpoint currently returns JSON, so there is no JSON API for a separate front-end or mobile client to consume.

```
Browser
   │  HTML forms (POST, plus ?_method=PUT/DELETE)
   ▼
app.js  ── helmet → session (Mongo store) → flash → passport → res.locals
   │
   ├── routes/*.js        URL mapping + middleware chains
   │      ├── middleware.js   isLoggedIn · isOwner · isReviewAuthor · saveRedirectUrl
   │      ├── schema.js       Joi validation (listing, review)
   │      └── multer + cloudConfig.js   image upload → Cloudinary
   ▼
controllers/*.js      business logic
   │      ├── Mapbox Geocoding   (create / update listing)
   │      └── Google Gemini      (review summary on listing page)
   ▼
models/*.js (Mongoose) ──► MongoDB
   │
   ▼
views/*.ejs  (layout: views/layouts/boilerplate.ejs)  → HTML to browser
```

**Request lifecycle (`app.js`)**
1. `.env` is loaded with `dotenv` unless `NODE_ENV=production`.
2. Mongoose connects to `ATLASDB_URL`.
3. Body parsers, `method-override` (`?_method=`), static files from `public/`.
4. `helmet` (CSP off), session middleware backed by MongoDB, `connect-flash`, Passport.
5. A middleware exposes `successMsg`, `errorMsg`, `currUser` and `query` (the `q` search term) to every view.
6. Routers are mounted (see [Routes](#routes-reference)); any unmatched URL becomes a `404 "Page Not Found!"` `ExpressError`.
7. The final error handler renders `views/listings/error.ejs` with the error message.

**Async errors:** async route handlers are wrapped with `utils/wrapAsync.js`, forwarding rejections to the error handler. `utils/ExpressError.js` carries a `statusCode` and `message`.

**Authorization**
- `isLoggedIn`: redirects to `/login` (remembering `req.originalUrl`) when unauthenticated.
- `isOwner`: only the listing's owner may edit or delete it.
- `isReviewAuthor`: only the review's author may delete it.

---

## Folder Structure

```
RentNow-fullstack/
├── app.js                  # App entry point: config, middleware, routers, error handling, server start
├── cloudConfig.js          # Cloudinary configuration + Multer storage engine
├── middleware.js           # isLoggedIn, saveRedirectUrl, isOwner, isReviewAuthor
├── schema.js               # Joi schemas: listingSchema, reviewSchema
├── package.json            # Dependencies and npm scripts
├── package-lock.json
├── vercel.json             # Vercel build/routing config (routes everything to app.js)
├── .gitignore              # Ignores .env and node_modules/
│
├── controllers/            # Request handlers (business logic)
│   ├── listings.js         #   index/search/filter, show (+Gemini summary), create/update (+Mapbox), delete
│   ├── reviews.js          #   create / delete review
│   ├── bookings.js         #   booking form + create booking (cost calc)
│   ├── dashboard.js        #   dashboard hub, my bookings, my listings
│   └── users.js            #   signup, login, logout (+ a legacy dashboard handler)
│
├── models/                 # Mongoose models
│   ├── listing.js          #   Listing (+ post-delete hook removing its reviews)
│   ├── review.js           #   Review
│   ├── booking.js          #   Booking
│   └── user.js             #   User (passport-local-mongoose plugin)
│
├── routes/                 # Express routers
│   ├── listing.js          #   /listings
│   ├── review.js           #   /listings/:id/reviews
│   ├── bookings.js         #   /listings/:listingId/bookings
│   ├── dashboard.js        #   /dashboard
│   └── user.js             #   /signup, /login, /logout
│
├── utils/
│   ├── wrapAsync.js        # Async error wrapper
│   └── ExpressError.js     # Custom error class
│
├── views/                  # EJS templates
│   ├── layouts/boilerplate.ejs      # Base layout (CDN assets, navbar, flash, footer)
│   ├── includes/                    # navbar, footer, flash partials
│   ├── listings/                    # index, show, new, edit, error
│   ├── bookings/new.ejs             # Booking form with live price preview
│   ├── dashboard/                   # main, bookings, listings
│   ├── users/                       # login, signup
│   └── legal/                       # privacy, terms
│
├── public/                 # Static assets served at /
│   ├── css/style.css       #   Site styles
│   ├── css/rating.css      #   Star rating widget styles
│   ├── js/script.js        #   Bootstrap client-side form validation
│   └── js/map.js           #   Mapbox GL map on the listing page
│
└── init/                   # Database seeding
    ├── index.js            #   Seed script (wipes and re-inserts listings)
    └── data.js             #   29 sample listings
```

---

## Data Models

**User** (`models/user.js`): `email` (required) plus `username`, `hash`, `salt` added by `passport-local-mongoose`.

**Listing** (`models/listing.js`)

| Field | Type | Notes |
|---|---|---|
| `title` | String | required |
| `description` | String | |
| `image` | `{ url, filename }` | Cloudinary URL and public ID |
| `price` | Number | per night |
| `location`, `country` | String | |
| `reviews` | `[ObjectId → Review]` | |
| `owner` | `ObjectId → User` | |
| `geometry` | GeoJSON `Point` | `coordinates: [lng, lat]`, from Mapbox |
| `category` | String enum | `Trending`, `Rooms`, `Iconic Cities`, `Mountains`, `Castles`, `Amazing Pools`, `Camping`, `FarmHouse`, `Arctic`, `Boats`, `Domes`, `Others` |

Deleting a listing via `findByIdAndDelete` also deletes its reviews (post-hook).

**Review** (`models/review.js`): `comment` (String), `rating` (Number 1–5), `createdAt`, `author` (`ObjectId → User`).

**Booking** (`models/booking.js`): `listing`, `user` (both required ObjectIds), `startDate`, `endDate`, `totalCost` (all required), `createdAt`.

**Validation (Joi, `schema.js`)**
- Listing: `title`, `description`, `location`, `country`, `category` required strings; `price` number ≥ 0; `image` optional string.
- Review: `rating` number 1–5 and `comment` string, both required.

---

## Routes Reference

| Method | Path | Auth | Description |
|---|---|---|---|
| GET | `/` | – | Redirects to `/listings` |
| GET | `/listings` | – | All listings; optional `?category=` and `?q=` |
| POST | `/listings` | Login | Create listing (multipart, field `listing[image]`) |
| GET | `/listings/new` | Login | New listing form |
| GET | `/listings/:id` | – | Listing detail, reviews, AI summary, map |
| GET | `/listings/:id/edit` | Owner | Edit form |
| PUT | `/listings/:id` | Owner | Update listing (via `?_method=PUT`) |
| DELETE | `/listings/:id` | Owner | Delete listing (via `?_method=DELETE`) |
| POST | `/listings/:id/reviews` | Login | Add review |
| DELETE | `/listings/:id/reviews/:reviewId` | Review author | Delete review |
| GET | `/listings/:listingId/bookings/new` | Login | Booking form |
| POST | `/listings/:listingId/bookings` | Login | Create booking |
| GET | `/dashboard` | Login | Dashboard hub |
| GET | `/dashboard/bookings` | Login | Your bookings |
| GET | `/dashboard/listings` | Login | Your listings |
| GET / POST | `/signup` | – | Registration form / submit |
| GET / POST | `/login` | – | Login form / submit |
| GET | `/logout` | – | Log out |
| GET | `/privacy`, `/terms` | – | Static legal pages |

---

## Prerequisites

- **Node.js 18.20.4** (the version declared in `package.json`) and npm
- **MongoDB**: a local instance or a MongoDB Atlas cluster
- A **Cloudinary** account (image uploads)
- A **Mapbox** account with a *public* access token (geocoding and maps)
- A **Google Gemini API key** (optional in practice: only needed for AI review summaries; without it the listing page shows a fallback message once a listing has 3+ reviews)

---

## Installation & Setup

```bash
# 1. Clone
git clone https://github.com/JithinManjunadh/RentNow-fullstack.git
cd RentNow-fullstack

# 2. Install dependencies
npm install

# 3. Create your environment file (see the next section for each value)
#    .env is git-ignored. Create it in the project root.

# 4. Start the app in development mode
npm run dev
```

Then open **http://localhost:3000**.

The port is hard-coded to `3000` in `app.js` (there is no `PORT` variable).

---

## Environment Variables

Create a `.env` file in the project root. The file is listed in `.gitignore`; **never commit it**.

```env
# MongoDB connection string (Atlas or local, e.g. mongodb://127.0.0.1:27017/wanderlust)
ATLASDB_URL=<your-mongodb-connection-string>

# Secret used to sign session cookies and encrypt the session store. Use a long random string.
SECRET=<long-random-string>

# Cloudinary (https://cloudinary.com/console)
CLOUD_NAME=<cloudinary-cloud-name>
CLOUD_API=<cloudinary-api-key>
CLOUD_API_SECRET=<cloudinary-api-secret>

# Mapbox PUBLIC token (starts with "pk."). It is also injected into the listing page for the browser map.
MAP_TOKEN=<mapbox-public-access-token>

# Google Gemini. `new GoogleGenAI({})` in controllers/listings.js has no explicit key,
# so the SDK reads it from the environment (GEMINI_API_KEY or GOOGLE_API_KEY).
GEMINI_API_KEY=<gemini-api-key>
```

| Variable | Used in | Purpose |
|---|---|---|
| `ATLASDB_URL` | `app.js` | Mongoose connection **and** session store |
| `SECRET` | `app.js` | Session secret and session-store encryption |
| `CLOUD_NAME`, `CLOUD_API`, `CLOUD_API_SECRET` | `cloudConfig.js` | Cloudinary credentials |
| `MAP_TOKEN` | `controllers/listings.js`, `views/listings/show.ejs` | Mapbox geocoding (server) and map rendering (browser) |
| `GEMINI_API_KEY` / `GOOGLE_API_KEY` | `@google/genai` SDK (read implicitly) | Review summaries |
| `NODE_ENV` | `app.js` | When `production`, `.env` is **not** loaded (set variables in the host instead) |

Because `MAP_TOKEN` is rendered into the HTML of the listing page, use a Mapbox **public** token (restrict it by URL in your Mapbox dashboard), never a secret one.

---

## Commands

| Command | What it does |
|---|---|
| `npm install` | Install dependencies |
| `npm start` | Run `node app.js` |
| `npm run dev` | Run `nodemon app.js` (auto-restart on changes) |
| `node init/index.js` | Seed sample listings (see below) |

> `nodemon` is not declared in `package.json`. If `npm run dev` reports "nodemon: not found", install it with `npm install -g nodemon` or `npm install --save-dev nodemon`, or use `npm start`. There is no test suite, linter or build script.

---

## Seeding Sample Data

`init/data.js` contains 29 sample listings (with images from Unsplash and pre-set coordinates). `init/index.js` loads them.

**Read before running:** the script runs `Listing.deleteMany({})`, which **deletes every listing** in the target database before inserting the samples.

Current behaviour of `init/index.js`:
- It connects to a **hard-coded** `mongodb://127.0.0.1:27017/wanderlust` (the `ATLASDB_URL` lines are commented out).
- It assigns every sample listing a **hard-coded `owner` ObjectId**.
- It does not close the connection when finished (press `Ctrl+C` after you see `data was initialized`).

Because the listing page reads `listing.owner._id`, seeded listings need an owner that actually exists in your database. Suggested flow:

1. Start MongoDB locally and set `ATLASDB_URL=mongodb://127.0.0.1:27017/wanderlust` in `.env`.
2. Run the app, sign up a user, and copy that user's `_id` from the `users` collection (e.g. with MongoDB Compass or `mongosh`).
3. In `init/index.js`, replace the hard-coded owner ID with your user's `_id` (this is a local edit for your own environment; do not commit your IDs).
4. Run `node init/index.js`, then `Ctrl+C`.

---

## Usage Walkthrough

1. **Sign up** at `/signup`, then browse `/listings`.
2. **Search / filter** using the navbar search box or the category icons.
3. **Add a listing** via *Add New Listing* (login required): title, description, category, image, price, country and location (e.g. `Goa, India`). The location is geocoded to place the map marker.
4. **View a listing** to see its details, reviews, AI summary (3+ reviews) and map. Owners see Edit and Delete buttons.
5. **Book** with *Book This Place*, pick dates, and confirm. A success flash shows the total cost.
6. **Review** a listing you don't own, with a star rating and comment.
7. **Dashboard** (`/dashboard`) shows *My Bookings* and *My Listings*.

---

## Deployment

The repo includes a `vercel.json` that builds `app.js` with `@vercel/node` and routes all requests to it. For any host:

- Set all [environment variables](#environment-variables) in the hosting dashboard and set `NODE_ENV=production`.
- Use a hosted MongoDB (e.g. Atlas) for `ATLASDB_URL`, and allow your host's IPs in Atlas network access.
- Note that `app.js` calls `app.listen(3000)` unconditionally and the `module.exports = app` line is commented out. A serverless platform such as Vercel normally expects an exported handler, so check that this setup works for your target host before relying on it.

---

## Known Issues & Caveats

These were found while reading the code. They are documented here, not fixed.

**Category values are inconsistent** (affects filtering and saving)
- The model enum uses `Iconic Cities`, `Amazing Pools`, `FarmHouse`. The index page filter links and the *edit* form use `IconicCities` and `AmazingPools`, so those two filter links match no listings, and saving an edit with those values fails Mongoose enum validation.
- The *new listing* form offers `Farm House` (with a space), which does not match the enum value `FarmHouse`.
- The *edit* form's "Others" option has `value="Boats"`, and `Domes` is in the enum and seed data but has no filter link or form option.

**Code bugs**
- `controllers/reviews.js`: the owner-review guard redirects to an undefined `listingId` variable (the UI hides the form from owners, so it's only reachable by a crafted request).
- `controllers/users.js`: `next` is used in the `req.login` error path but isn't defined in `signup`.
- `routes/user.js` defines `GET /dashboard` rendering `users/dashboard.ejs`, which doesn't exist; the route is shadowed by the `/dashboard` router mounted earlier in `app.js`.
- `models/review.js`: `createdAt` uses `Date.now()` (called) as the default, so every review gets the server's start time.
- `views/listings/show.ejs`: the AI summary box is hidden only when the summary equals `"No reviews yet."`, but the controller's default text is different, so the box shows the default message even for listings with fewer than 3 reviews.
- Listing price on the *new* form is a plain text input; the Joi schema enforces a number.

**Product limitations (not implemented)**
- No check for overlapping bookings or availability, no cancellation, no payments. Booking names/emails posted by the form are not stored; only listing, user, dates and total cost are.
- Dashboard has no delete/cancel actions; the "+ 18% GST" toggle is a visual label only.
- Logout is a `GET` request.
- `helmet` is configured with `contentSecurityPolicy: false`.
- `cookie-parser` is installed but not used anywhere in the code.
- The Gemini request logs the review text to the server console.

---

## Contributing Tips

- Follow the existing pattern when adding a feature: **model → controller → route (+ middleware/validation) → view**.
- Wrap async handlers in `wrapAsync` and throw `ExpressError(status, message)` for errors that should reach the error page.
- Add Joi rules in `schema.js` for any new form input.
- Never commit `.env`, API keys or database IDs tied to your environment.