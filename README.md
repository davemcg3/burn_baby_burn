# Burn Baby Burn 🔥 Meeting Inferno

A meeting burn clock calculator / timer.

Made while _in a meeting_, modified while _in multiple other meetings_.

Visit https://davemcg3.github.io/burn_baby_burn/ for this version. Deployed with github pages. 

Notable changes from the original include a rewrite of the JavaScript structure, improvements to the mobile view so that it's usable on mobile, improvements (IMO) to the user interface while keeping consistent with the original theme, an improvement to break meetings into increments so that you can add and remove people from the meeting counter as they join and leave the meeting to get a more accurate cost, and an addition of using localStorage to store details on the latest few meetings. 

It's vanilla JavaScript so you just need a web server to use this locally.

## Running Locally

### Using Docker (Recommended)

Build and run using Docker Compose:
```bash
docker-compose up -d
```

The site will be available at `http://localhost:8080`.

To stop the container:
```bash
docker-compose down
```

Alternatively, build and run with Docker directly:
```bash
docker build -t burn-baby-burn .
docker run -d -p 8080:80 burn-baby-burn
```

### Using a Local Web Server

Since this is vanilla JavaScript, you can use any web server. For example:

**Python 3:**
```bash
python3 -m http.server 8000
```

**Node.js (with http-server):**
```bash
npx http-server -p 8000
```

**PHP:**
```bash
php -S localhost:8000
```

Then open `http://localhost:8000` in your browser.

Photo by Greyson Joralemon on Unsplash
