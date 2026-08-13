# CTF Notebook — Setup Guide

This is a [Jekyll](https://jekyllrb.com/) site, which GitHub Pages builds
and hosts automatically for free. You don't need to run a build step
yourself in production — you just push markdown files and GitHub does the
rest. You only need Jekyll installed locally if you want to preview posts
before pushing (recommended, but optional).

## 1. Create the GitHub repo

1. Go to github.com and create a new repository named exactly:
   `yourusername.github.io` (replace `yourusername` with your actual GitHub
   username — this exact naming is what makes GitHub Pages auto-publish it).
2. Don't initialize it with a README — you already have files here.

## 2. Push this scaffold to GitHub

From this folder, in your terminal (VS Code terminal is fine):

```bash
git init
git add .
git commit -m "initial site scaffold"
git branch -M main
git remote add origin https://github.com/yourusername/yourusername.github.io.git
git push -u origin main
```

## 3. Enable GitHub Pages

1. On GitHub, go to your repo → **Settings** → **Pages**.
2. Under "Build and deployment", source should be **Deploy from a branch**,
   branch **main**, folder **/ (root)**.
3. Save. Your site will be live at `https://yourusername.github.io` within
   a minute or two.

## 4. Update `_config.yml`

Open `_config.yml` and replace:
- `url:` with `"https://yourusername.github.io"`
- the `github.user_url` under `social_links` with your actual GitHub profile

## 5. Preview locally (optional but recommended)

Since you already have Homebrew:

```bash
brew install ruby
gem install bundler
bundle install
bundle exec jekyll serve
```

Then open `http://localhost:4000` in your browser. Any time you save a
file, Jekyll rebuilds automatically — just refresh the page.

## Writing a new post

1. Copy `_writeup_template.md` into the `_posts/` folder.
2. Rename it to match Jekyll's required format: `YYYY-MM-DD-short-title.md`
   (the date in the filename controls the post's sort order and URL).
3. Fill in the front matter (`title`, `date`, `categories`, `tags`) and the
   body sections.
4. Commit and push — it's live once GitHub Pages rebuilds (~1 min).

```bash
git add _posts/2026-08-15-my-new-writeup.md
git commit -m "writeup: my new writeup"
git push
```

## Folder structure

```
ctf-blog/
├── _config.yml           site settings
├── _posts/                published writeups (filename = date + title)
├── _drafts/                work-in-progress posts (not published, no date needed)
├── _writeup_template.md   copy this into _posts/ for each new writeup
├── writeup-template.md    same template, rendered as a page on the live site
├── index.md               homepage
└── Gemfile                local preview dependencies
```

## Notes

- Posts in `_drafts/` don't need a date in the filename and won't publish —
  useful for a challenge you're still working through. Preview drafts
  locally with `bundle exec jekyll serve --drafts`.
- `categories` in front matter feeds into the URL structure
  (`/pwn/2026/08/12/title/`), so keep them short and consistent
  (`pwn`, `crypto`, `forensics`, `web`, `reverse-engineering`, `misc`).
