# Blog writing voice

Posts must sound like Daniil, not like a polished tech magazine. Reference posts for the real voice:
`content/posts/bad-hr-experience-or-how-new-repo-was-born`, `how-i-became-digital-nomad-cto`,
`wfh-wtf`, `single-file-chat-in-elixir-phoenix`, `connect-livebook-to-phoenix-on-vps`.
Read at least two of them before drafting. Do NOT use the 2026-09 and 2026-10 posts as a reference: readers called them AI-written and boring.

## How he writes

- Starts from a personal reason: "I had insomnia and built it in one night", "I always wanted something quick to test LiveView prototypes". The story comes first, the tech after.
- First person, talking to the reader. "Let me tell about it", "Let's jump into", "What's the plan?", "What it does?".
- Honest and self-deprecating: "I'm a backend developer and you will notice it from my frontend skills", "not optimal, only for quick night dirty prototyping".
- Has opinions and says them plainly, sometimes sharp (see `wfh-wtf`). No "on the other hand" balance just for balance.
- Short. Most good posts are 300 to 1500 words. One idea per post. Split a talk into several posts instead of one 4000-word essay.
- Short plain sentences, contractions, a bit of non-native English rhythm. Keep it. Do not polish it into perfect grammar.
- Asides as blockquotes with an emoji: `> ⁉️ ...`, `> ❤️ Huge thanks to ...`. Emoji are rare but welcome.
- Concrete before abstract: real file, real command, real number, real mistake. Code he actually ran.
- Keep measurement tables (timings, sizes, benchmarks, device/protocol specs). He likes them; when shortening a post, cut prose, not numbers.
- Ends simple: a link to the repo, "feel free to leave a comment", "not sure if it will be useful to anyone, but I decided to share it". No grand conclusion.

## Never do this (the reason the last two posts read as AI)

- "This is not X, it is Y." / "That is not a coincidence, it is ..." / "This isn't an Elixir pitch."
- Rhetorical drama lines: "Color science was not in the job description. It became one anyway."
- Groups of three for rhythm: "camera in log, color box, output".
- Inflated words: genuinely, tractable, deliberate, seamless, crucial, robust, landscape, delve, "the entire reason", "the one rule that matters".
- Every paragraph the same length, every section ending with a neat punchline.
- Em-dashes. Use a comma, a period, or brackets.
- Colon setups and clever framing: "My answer was ...:", "That's exactly what I liked about it:", "So X is just a number:". Just say it: "So I made ...".
- Punchy repeat lines: "Not hard, gently." / "and then it's a different project." / quoted mini-slogans like "thinking of you from the other room".
- Generic hero intros ("Quick context if you're new here: I own the software side of...") repeated across posts.
- Explaining textbook theory at length (cones, wavelengths). Link to it and get to what you built and what broke.
- A literary pastiche opening (the Fear and Loathing riff in `elixir-liveview-single-binary`) followed by a dry generated guide.

## Process

1. Ask Daniil for the story: why he built it, what annoyed him, what broke, what he'd do differently. Use his words.
2. Draft short. Show him. He adds the jokes and the opinions, do not invent them for him.
3. Before saving, run the `humanizer` skill over the draft and re-check the "never" list above.
