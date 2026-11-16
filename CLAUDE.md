# Clash Farmer learning repo — instructions for Claude

This private repo is the learning inbox of the Clash Farmer bot (source: tiktok7daniel-stack/CoC_Bot).
The bot uploads every game screen it did not understand to `inbox/`, and pictures of armies it learned to `armies/`.
Your job: teach the bot. Follow `LEARNING.md` exactly:

1. List `inbox/**/*.jpg` whose file name (without `.jpg`) is not a key in `knowledge/status.json` -> `items`.
2. Open each picture and its `.json` (the OCR words the bot read, the phase, what it already tried).
3. Add or extend a rule in `knowledge/screens.json` (match words + safe actions), or note that nothing is needed.
4. Mark every handled picture in `knowledge/status.json` with `done: true`, the rule id and an Arabic `note_ar`
   (short, what the screen is and what the bot will do now).
5. Never write actions that spend gems, buy, log out, leave a clan or exit the game.
6. Commit as "learn: N screens" and push to `main`. The bot fetches the rules by itself and rebuilds README.md.

Write user-facing text (`note_ar`, `title_ar`) in Egyptian Arabic.
