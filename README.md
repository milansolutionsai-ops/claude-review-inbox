# claude-review-inbox: review what your AI agent made without reading its chat

Claude Code builds things for you. Then you review them by scrolling the chat, typing
"at 0:14 the cut is too early, and the caption at the end is wrong", and hoping it
understood which 0:14 you meant. The next round, you can't tell what changed.

This is a small local app that gives the review its own place. Claude posts its work.
You mark the exact moment, write a note, and press Send. On your next message, Claude is
told "Feedback ready: 2 notes" without you pasting anything. It fixes them and posts
round 2, with one line per note on what it changed. You then check each fix
before and after, side by side.

I built it to review AI-edited videos, so it is built around video renders. The pattern
fits any agent work you judge by looking at the result, not by reading the code.

## What you get

- **An inbox.** Every piece of work waiting for you, with its status: waiting for you,
  sent, fixes ready, approved.
- **Notes pinned to exact frames.** Click or drag on the timeline, type the note. The
  agent gets the note plus stills of those exact frames, so "too early" means one thing.
- **Feedback reaches the agent by itself.** A Claude Code hook reads one index file
  and tells the agent when you've sent something. No copy-paste.
- **Fix check.** Round 2 arrives with the agent's answer to every note and a
  before/after for each. You mark each one Fixed or Not fixed.
- **Plan first (optional).** For bigger jobs the agent posts a timed plan, line by line,
  before building anything. You OK each line or say what to change.
- **Local and private.** Runs on 127.0.0.1 only. Python standard library plus ffmpeg,
  and Node for the hook. No accounts, no uploads, no paid APIs.

## 60-second quickstart

You need Python 3.10+, ffmpeg on your PATH, and Node 18+ (for the hook).

```bash
git clone https://github.com/milansolutionsai-ops/claude-review-inbox
cd claude-review-inbox
python server.py
```

Open http://127.0.0.1:47615. Leave the server running and, in a second terminal in the
same folder, make a 10-second test clip and send it for review:

```bash
ffmpeg -f lavfi -i testsrc=duration=10:size=1280x720:rate=30 -f lavfi -i sine=duration=10 -shortest -pix_fmt yuv420p demo.mp4
python review.py register --project demo --title "First test" --render demo.mp4
```

Refresh the page: "First test" is waiting for you. Drop a note on a frame and press Send.
Then run the agent's side yourself to see what Claude would get:

```bash
python review.py claim demo/first-test
python review.py show demo/first-test
```

That's the whole loop. The next two steps wire it into Claude Code.

## Wire it into Claude Code

**1. Where your work lives.** The app keeps everything under one workspace folder,
`REVIEW_ROOT`. By default that's the folder you cloned this repo into, so clone it next to
your projects, or set `REVIEW_ROOT` to the folder that holds them. Set it where both the
server and Claude Code can see it.

**2. The hook.** Add this to `~/.claude/settings.json`, using the real path to your clone:

```json
{
  "hooks": {
    "UserPromptSubmit": [
      { "hooks": [{ "type": "command", "command": "node \"/path/to/claude-review-inbox/hook/announce.mjs\"", "timeout": 5 }] }
    ]
  }
}
```

The hook only reads `_review-index.json`. It stays silent outside your workspace and never
blocks a prompt.

**3. Tell Claude the loop.** Paste this into your project's `CLAUDE.md`:

```markdown
## Review loop
When a render or other visual result is ready for me, send it for review instead of describing it in chat:
python /path/to/claude-review-inbox/review.py register --project <project> --title "<title>" --render <file.mp4>
When the hook says "Feedback ready", run the claim and show commands it prints, fix every note, then register
round 2 with --fixes fixes.json (one plain-words line per note).
```

On Windows, `python install.py` also starts the server with Windows and opens the app in
Edge, where you can install it as an app with its own taskbar icon. It prints the hook
block but never edits your Claude settings.

## The agent's commands

```bash
python review.py register --project my-project --title "My video" --render final.mp4
python review.py list
python review.py claim my-project/my-video
python review.py show my-project/my-video
python review.py register --project my-project --title "My video" --render v2.mp4 --fixes fixes.json
```

`fixes.json` answers every open note (new notes, plus anything you marked Not fixed):

```json
{"answers": [{"note_id": "ab12cd", "explanation": "Moved the cut 0.4s later.", "after": [412, 440]}]}
```

`after` is optional. It's the frame range to show as "after" when the edit moved the spot.

### Plan first

```bash
python review.py plan --project my-project --title "My video" --plan plan.json
```

```json
{"rows": [{"start": 0.0, "end": 2.8, "text": "Hook: face, the word HONEST appears", "still": "s1.jpg"}]}
```

Times are seconds on the finished timeline and `still` is optional. You mark every line OK
or Change, with a reason. The hook then prints "Edit plan back", and the agent either
revises the plan (the old version is kept) or registers the first cut.

### Style rounds

`style.py` runs a side-by-side taste test. The agent posts several versions that each change
one thing (caption style, transition, music), and you sort them into Keep, Maybe and Drop.
`python style.py --help` has the commands.

## Where things are stored

```
<REVIEW_ROOT>/
  _review-index.json          one-file summary the hook reads
  <project>/_review/<slug>/
    video.json                title, status, rounds
    rN.mp4                    review copy of round N (your original file is never touched)
    rN/                       timeline sprite, waveform, stills
    notes-rN.json             your notes and verdicts for round N
    fixes-rN.json             the agent's answers that came with round N
    plan.json                 the edit plan, if one was posted
```

Plain JSON files, so you can read, back up or delete anything by hand.
The page only shows titles, video, notes and statuses, never file names or paths.

## Tests

```bash
python -m unittest discover -s tests -t .
node --test "**/*.test.mjs"
```

## License

MIT
