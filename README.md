# Source repository for blog.lespaulstudioplus.info

Content source consumed by the self-hosted blog API
(https://github.com/mhoshi-vm — Spring Boot app). Only three directories matter:

- `content/` — posts as markdown with YAML frontmatter (`*.md` = Japanese, `*.en.md` = English) and the about page
- `data/` — `menu.yaml` navigation
- `static/` — images served at `/images/**`

The Hugo site machinery that used to live here was removed on this branch;
the API renders markdown itself.
