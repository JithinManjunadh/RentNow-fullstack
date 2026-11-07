# RentNow

RentNow is a full-stack property rental web app. Hosts list places to stay, guests search, book and review them, and an AI summary condenses a listing's reviews.

Built with Node.js, Express, MongoDB, EJS, Cloudinary, Mapbox and Google Gemini.

---

## Features

- **Accounts:** sign up, log in and log out. Sessions are stored in MongoDB and last 7 days. If you open a protected page while logged out, you are sent to login and then back to that page.
- **Listings:** create, edit and delete listings with a photo, price per night, category and location. Only the owner can edit or delete a listing.
- **Search and filters:** search by title, location or country, and filter by category.
- **Maps:** a listing's location is converted to coordinates (Mapbox) and shown on a map on its page.
- **Bookings:** pick start and end dates; the total cost (nights × price) is calculated and saved. You cannot book your own listing.
- **Reviews:** 1–5 star rating plus a comment. Only the author can delete a review, and owners cannot review their own listing.
- **AI review summary:** when a listing has 3 or more reviews, Google Gemini writes a short summary shown on the listing page.
- **Dashboard:** "My Bookings" (stays you booked) and "My Listings" (places you host).

---

## Tech Stack

| Area | Technology |
|---|---|
| Server | Node.js 18.20.4 (as set in `package.json`), Express 5 |
| Views | EJS with `ejs-mate` layouts, Bootstrap 5, Font Awesome |
| Database | MongoDB with Mongoose |
| Login | Passport (`passport-local`, `passport-local-mongoose`) |
| Sessions and messages | `express-session`, `connect-mongo`, `connect-flash` |
| Validation | Joi |
| Image upload | Multer, Cloudinary |
| Maps | Mapbox (geocoding on the server, map in the browser) |
| AI | Google Gemini (`@google/genai`, model `gemini-2.5-flash`) |
| Other | `helmet`, `method-override`, `dotenv` |
| Deployment config | Vercel (`vercel.json`) |

---

## How It Works

RentNow is a server-rendered app: the browser requests a page, Express runs the logic, and EJS returns finished HTML. There is no separate front-end project.

```
Browser  →  routes  →  middleware (login / owner checks, validation, image upload)
                          →  controller (the logic)  →  model (MongoDB)
                          →  view (EJS)  →  HTML back to the browser
```

Forms can only send GET and POST, so edit and delete forms use `?_method=PUT` and `?_method=DELETE`, which `method-override` converts into the real HTTP methods.

---

## Project Structure (block by block)

```
RentNow/
├── app.js
├── cloudConfig.js
├── middleware.js
├── schema.js
├── package.json
├── vercel.json
├── controllers/
├── models/
├── routes/
├── utils/
├── views/
├── public/
└── init/
```

### Root files

| File | What it does |
|---|---|
| `app.js` | Starts the app. Loads `.env`, connects to MongoDB, sets up EJS, sessions, flash messages, Passport login and `helmet`, mounts all routers, handles 404s and errors, and listens on port 3000. It also makes `currUser`, success/error messages and the search text available in every view. |
| `cloudConfig.js` | Connects to Cloudinary and defines where uploaded images are stored (only png, jpg and jpeg are accepted). |
| `middleware.js` | Reusable checks: `isLoggedIn` (redirects to login and remembers the page), `saveRedirectUrl` (returns the user to that page after login), `isOwner` (only the listing owner may continue) and `isReviewAuthor` (only the review author may continue). |
| `schema.js` | Joi rules that validate listing and review form data before it is saved. |
| `package.json` | Dependencies and scripts (`npm start`, `npm run dev`). |
| `vercel.json` | Tells Vercel to run `app.js` and send every request to it. |

### `routes/` — which URL goes where

Each file links URLs to middleware and a controller function.

| File | Mounted at | Purpose |
|---|---|---|
| `listing.js` | `/listings` | List, create, show, edit, update and delete listings. Includes the image upload and validation steps. |
| `review.js` | `/listings/:id/reviews` | Add and delete reviews. |
| `bookings.js` | `/listings/:listingId/bookings` | Booking form and creating a booking. |
| `dashboard.js` | `/dashboard` | Dashboard home, my bookings, my listings. |
| `user.js` | `/` | Signup, login and logout. |

### `controllers/` — the logic

| File | What it does |
|---|---|
| `listings.js` | **Index:** applies the category filter and the search text. **Show:** loads the listing with its reviews and owner, and asks Gemini for a summary when there are 3 or more reviews. **Create / Update:** converts the location to coordinates with Mapbox and saves the Cloudinary image. **Delete:** removes the listing. |
| `reviews.js` | Creates a review (linked to the listing and the logged-in user) and deletes one. |
| `bookings.js` | Shows the booking form and creates a booking. It blocks booking your own listing, requires the end date to be after the start date, and calculates `totalCost`. |
| `dashboard.js` | Renders the dashboard and fetches the user's bookings and listings. |
| `users.js` | Registers a user, logs in and out, and shows flash messages. |

### `models/` — the database

| File | Stores |
|---|---|
| `user.js` | `email`, plus username and hashed password (added by `passport-local-mongoose`). |
| `listing.js` | `title`, `description`, `image` (url, filename), `price`, `location`, `country`, `category`, `geometry` (map coordinates), `owner` and `reviews`. Deleting a listing also deletes its reviews. |
| `review.js` | `comment`, `rating` (1–5), `author`, `createdAt`. |
| `booking.js` | `listing`, `user`, `startDate`, `endDate`, `totalCost`, `createdAt`. |

Listing categories: Trending, Rooms, Iconic Cities, Mountains, Castles, Amazing Pools, Camping, FarmHouse, Arctic, Boats, Domes, Others.

### `utils/`

| File | What it does |
|---|---|
| `wrapAsync.js` | Wraps async route handlers so errors reach the error handler instead of crashing. |
| `ExpressError.js` | Error class carrying an HTTP status code and a message. |

### `views/` — the pages (EJS)

| Folder / file | Page |
|---|---|
| `layouts/boilerplate.ejs` | Base layout used by every page: loads Bootstrap, Font Awesome, fonts and Mapbox, and includes the navbar, flash messages and footer. |
| `includes/` | `navbar.ejs` (links, search box, login state), `flash.ejs` (success/error alerts), `footer.ejs`. |
| `listings/` | `index.ejs` (all listings, category filter bar, GST display toggle), `show.ejs` (details, book button, reviews, AI summary, map), `new.ejs`, `edit.ejs`, `error.ejs` (shown for any error or 404). |
| `bookings/new.ejs` | Booking form with a live total price. |
| `dashboard/` | `main.ejs` (home), `bookings.ejs` (my bookings), `listings.ejs` (my listings). |
| `users/` | `login.ejs`, `signup.ejs`. |
| `legal/` | `privacy.ejs`, `terms.ejs`. |

### `public/` — files sent to the browser as-is

| File | Purpose |
|---|---|
| `css/style.css` | Site styling: navbar, footer, listing cards, buttons, map size. |
| `css/rating.css` | The star rating input and the star display on reviews. |
| `js/script.js` | Bootstrap form validation for forms marked `needs-validation`. |
| `js/map.js` | Draws the Mapbox map and a marker for the listing on its page. |

### `init/` — sample data

| File | Purpose |
|---|---|
| `data.js` | 29 sample listings. |
| `index.js` | Script that deletes all listings and inserts the samples (see [Sample Data](#sample-data)). |

---

## Routes

| Method | URL | Login needed | Purpose |
|---|---|---|---|
| GET | `/` | No | Redirects to `/listings` |
| GET | `/listings` | No | All listings (`?category=` and `?q=` supported) |
| GET | `/listings/new` | Yes | New listing form |
| POST | `/listings` | Yes | Create listing |
| GET | `/listings/:id` | No | Listing details |
| GET | `/listings/:id/edit` | Owner | Edit form |
| PUT | `/listings/:id` | Owner | Update listing |
| DELETE | `/listings/:id` | Owner | Delete listing |
| POST | `/listings/:id/reviews` | Yes | Add review |
| DELETE | `/listings/:id/reviews/:reviewId` | Review author | Delete review |
| GET | `/listings/:listingId/bookings/new` | Yes | Booking form |
| POST | `/listings/:listingId/bookings` | Yes | Create booking |
| GET | `/dashboard` | Yes | Dashboard home |
| GET | `/dashboard/bookings` | Yes | My bookings |
| GET | `/dashboard/listings` | Yes | My listings |
| GET, POST | `/signup` | No | Sign up |
| GET, POST | `/login` | No | Log in |
| GET | `/logout` | No | Log out |
| GET | `/privacy`, `/terms` | No | Legal pages |

---

## Getting Started

### Requirements

- Node.js 18.20.4 and npm
- A MongoDB database (local or MongoDB Atlas)
- A Cloudinary account
- A Mapbox public access token
- A Google Gemini API key (only needed for the AI summary)

### Install and run

```bash
git clone https://github.com/JithinManjunadh/RentNow-fullstack.git
cd RentNow-fullstack
npm install
```

Create a `.env` file in the project root (see below), then start the app:

```bash
npm run dev     # development, auto-restarts with nodemon
npm start       # plain node app.js
```

Open **http://localhost:3000**. The port is fixed to 3000 in `app.js`.

> `npm run dev` uses `nodemon`, which is not listed in `package.json`. If the command is not found, run `npm install -g nodemon` or just use `npm start`.

### Environment variables

`.env` is git-ignored. Never commit it.

```env
ATLASDB_URL=<your MongoDB connection string>
SECRET=<long random string for sessions>
CLOUD_NAME=<Cloudinary cloud name>
CLOUD_API=<Cloudinary API key>
CLOUD_API_SECRET=<Cloudinary API secret>
MAP_TOKEN=<Mapbox public token, starts with pk.>
GEMINI_API_KEY=<Google Gemini API key>
```

| Variable | Used for |
|---|---|
| `ATLASDB_URL` | Database connection and session storage (a local MongoDB URL works too) |
| `SECRET` | Signing and encrypting sessions |
| `CLOUD_NAME`, `CLOUD_API`, `CLOUD_API_SECRET` | Image uploads to Cloudinary |
| `MAP_TOKEN` | Location lookup and the map. It is also written into the listing page for the browser, so use a **public** token. |
| `GEMINI_API_KEY` | AI review summary. The code does not pass a key, so the Gemini SDK reads `GEMINI_API_KEY` (or `GOOGLE_API_KEY`) from the environment. |

When `NODE_ENV=production`, `.env` is not loaded, so set these variables in your hosting dashboard instead.

---

## Sample Data

To fill the site with 29 demo listings:

1. Run the app, sign up a user, and copy that user's `_id` from the `users` collection.
2. In `init/index.js`, replace the hard-coded owner ID with your user's `_id` (do not commit it).
3. Make sure `ATLASDB_URL` points to the same database that `init/index.js` connects to. The script has its own hard-coded local MongoDB URL.
4. Run `node init/index.js`, wait for `data was initialized`, then press `Ctrl+C` (the script does not exit on its own).

**Warning:** this script deletes every existing listing before inserting the samples.

---

## Deployment

`vercel.json` is included for Vercel. On any host, set the environment variables above, set `NODE_ENV=production`, and use a hosted MongoDB such as Atlas (allow your host's IP in Atlas network access).