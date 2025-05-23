# Mike's blog

[![Netlify Status](https://api.netlify.com/api/v1/badges/5da286f6-4221-457a-93c2-97ce0b6655c1/deploy-status)](https://app.netlify.com/sites/dreyfus/deploys)

## See the site directly

[dreyfus.netlify.app](https://dreyfus.netlify.app/)

## To use the template

- Connect to your chosen hosting provider (see Deploy to Netlify button below if you want to go that route, otherwise use the GitHub template button above and pick a different one)
- Make an account at [tina.io](https://tina.io/)
- Add your TinaCMS keys (see below)
- Update `astro.config.mjs` with your domain
- Edit `src/config.js`
- Add your URL in line 1 of `public/robots.txt`
- Add your links in `src/components/Header.astro`
- Update the intro in `pages/about.md`
- Edit the images in `public/` (optional)
- Edit whatever tags you want in `tina/config.js` (optional)

After this, you can add your content to `src/posts` with Markdown files, or with TinaCMS by going to `yoururl.com/admin`!

[![Deploy to Netlify](https://www.netlify.com/img/deploy/button.svg)](https://app.netlify.com/start/deploy?repository=https://github.com/michaeldrey/dreyfusblog)

This deploys to [dreyfus.dev](https://dreyfus.dev)

## Run it yourself

All commands are run from the root of the project, from a terminal:

| Command                          | Action                                                        |
| :------------------------------- | :------------------------------------------------------------ |
| `npm install`                    | Installs dependencies                                         |
| `npm run dev`                    | Starts local dev server at `localhost:4321`                   |
| `npx tinacms dev -c 'astro dev'` | Manually run local server if the regular command doesn't work |
| `npm run build`                  | Build your production site to `./dist/`                       |
| `npm run preview`                | Preview your build locally, before deploying                  |

You go to `localhost:4321/admin/index.html` to see the CMS and use it. If you want to clone this for yourself, you'll need a `.env.development` file that has the following in it:

```
TINACLIENTID=<from tina.io>
TINATOKEN=<from tina.io>
TINASEARCH=<from tina.io>
```

Credit: ![Thank you cassidoo!](https://github.com/cassidoo/blahg)
