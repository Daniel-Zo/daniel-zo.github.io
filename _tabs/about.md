---
# the default layout is 'page'
icon: fas fa-info-circle
order: 4
---

# Still figuring out how to introduce myself.

Every About page claims to have answers. This one is honest about not having them yet —
so here's a live placeholder that updates as I do.

**Hi, I'm Danile.** _a work in progress, shipping anyway._ ~~(this line rotates — see below)~~

---

## ✍️ Rotating one-liners (typewriter pool, pick 5–7)

**Universal**
- a work in progress, shipping anyway.
- less of a person, more of a changelog.
- an expert at starting new hobbies.
- still under construction. Thanks for your patience.
- 80% curiosity, 20% coffee.

**Developer-flavored**
- the friend who actually reads the error message.
- powered by coffee and merge conflicts.
- it works on my personality.
- commit early, commit often, regret nothing.
- my hobbies have a tech-debt problem.

**Literary / quiet**
- a rough draft that refuses the trash bin.
- mostly questions, occasionally answers.
- edited by life, revised daily.
- an unfinished sentence, looking for its period.

---

## 🗑️ Rejected drafts

| Draft | Why it was rejected |
|---|---|
| ~~"Visionary thought leader."~~ | too LinkedIn |
| ~~"Just a human being, you know?"~~ | too vague |
| ~~"Go-getter. People person. Team player."~~ | too corporate |
| ~~"Ask my mom, she'll tell you."~~ | biased source |
| ~~"[YOUR INDUSTRY'S FAKEST BUZZWORD PHRASE]"~~ | too corporate |

> Tip: the last row works best when it's the most clichéd sentence your industry actually uses.

---

## 📊 Self-knowledge

```
Self-knowledge  [███████████████░░░░░]  73%
```
remaining 27% under active investigation

---

## 📝 30-second fill-in version

> **Still figuring out how to introduce myself.**
> Hi, I'm **[NAME]**. I spend most days **[what you actually do, one sentence]**
> and the rest **[one real quirk]**.
>
> Rejected drafts: ~~"visionary thought leader"~~ *(too LinkedIn)* ·
> ~~"just a human being"~~ *(too vague)* · ~~"[industry buzzword]"~~ *(too corporate)*
>
> Self-knowledge: 73% — the rest is under active investigation.

---

## 🛠️ Implementation notes

**Typewriter rotation (vanilla JS)**
```js
const lines = [
  "a work in progress, shipping anyway.",
  // ...your 5-7 picks from the pool above
];
let i = 0, el = document.querySelector("#typed"), timer;
function type(str, cb) {
  el.textContent = ""; let p = 0;
  (function step() {
    if (p <= str.length) { el.textContent = str.slice(0, p++); timer = setTimeout(step, 45); }
    else if (cb) timer = setTimeout(cb, 1800);
  })();
}
function erase(cb) {
  (function step() {
    const s = el.textContent;
    if (s.length) { el.textContent = s.slice(0, -1); timer = setTimeout(step, 18); }
    else if (cb) cb();
  })();
}
function next() { erase(() => { i = (i + 1) % lines.length; type(lines[i], next); }); }
type(lines[0], next);
```

**Styling hints**
- Rejected drafts: `text-decoration: line-through` + a muted gray for the text, smaller gray label on the right.
- Progress bar: ~6px tall, full-width pill, solid dark fill — keep it monochrome.
- Add a "Shuffle intro" button for visitors (and for you, when you're bored).
- Page footer: `Last updated: just now · Next revision: whenever I figure it out`

**Footer**
```
Last updated: just now · Next revision: whenever I figure it out
```

