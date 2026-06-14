# Repo metadata

Plumbing for the repository itself, kept out of the top level so the public file
listing stays focused on the guides.

## Social preview image

GitHub shows a preview card when a repo or file URL is shared on social media
(OpenGraph / X). For `github.com` blob URLs the card is **repo-level**: the same
description and image apply to every file. To override GitHub's auto-generated
image, upload a custom one under **Settings → General → Social preview**.

GitHub stores the uploaded image on its own side, so these files are just the
versioned source — edit the SVG and re-export when the card needs to change.

| File                 | Purpose                                          |
| -------------------- | ------------------------------------------------ |
| `social-preview.svg` | Source of truth — edit this.                     |
| `social-preview.png` | 1280×640 export; the file uploaded to GitHub.    |

Notes:

- GitHub recommends **1280×640** (displayed at half size; min 640×320).
- Keep important content away from the edges — some clients crop to a 2:1 card.
