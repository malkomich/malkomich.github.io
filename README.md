[![Netlify Status](https://api.netlify.com/api/v1/badges/38ae83e3-75f3-4f94-8611-283c580b9ac0/deploy-status)](https://app.netlify.com/sites/malkomich/deploys)
[![Node](https://img.shields.io/badge/Node-13.5.0-informational.svg?style=for-the-badge&logo=node.js)](https://nodejs.org/es/)
[![NPM](https://img.shields.io/badge/NPM-6.14.4-informational.svg?style=for-the-badge&logo=npm)](https://www.npmjs.com/)
[![Jekyll](https://img.shields.io/badge/Jekyll-%3E%3D%203.8.6-informational.svg?style=for-the-badge&logo=jekyll&logoColor=critical)](https://jekyllrb.com/)
[![Jekflix](https://img.shields.io/badge/Jekflix-3.1.0-informational?style=for-the-badge&logo=github&logoColor=black)](https://github.com/thiagorossener/jekflix-template)

### Run steps

1. [Install Ruby with Dev Toolkit](https://rubyinstaller.org/downloads/) (Need permissions on Ruby installation folder)
2. *ONLY WINDOWS* - `ridk install`: Install [MSYS2](https://www.msys2.org/) if not done in previous step (Need permissions on the installation folder)
3. `npm i`: Install dependencies
4. `npm run build`: Build Jekyll blog
5. `npm run dev`: Run blog

### Create new Post

1. `npm run post "Your Post Title"`
2. The new post format is `YYYY-MM-DD-your-post-title.md`
3. Configure the properties, keeping them wrapped in `---` separators:
    * **date**: The post publishing date. Format: `YYYY-MM-DD hh:mm:ss`.
    * **layout**: Default to `post` for post entries.
    * **title**: The post title.
    * **subtitle**: A subtitle to appear below the title.
    * **description**: Used in the home and category pages, in meta description tag for SEO purposes and for social media sharing.
    * **image**: Featured image shown in post reading view.
    * **optimized_image** _(Optional, Recommended)_: Social preview image (LinkedIn/Open Graph) and card image for home/category pages. Recommended size: `1200x627` (ratio `1.91:1`).
    * **category**: One category defined in `category/<category>.md`.
    * **tags**: Post keywords. Used in the home, category and tags pages, and as meta keywords for SEO purposes.
    * **author**: One author defined in `_authors/<author>.md`.
    * **paginate**: To break your post into pages. Use the divider --page-break-- where you want to break it.
4. Write the post


### Theme & dependencies

- Based on the awesome [Jekflix](https://github.com/thiagorossener/jekflix-template) theme, by [@ThiagoRossener](https://github.com/thiagorossener)
- Particles animation for index page, based on [ParticlesJS](https://github.com/VincentGarreau/particles.js)
