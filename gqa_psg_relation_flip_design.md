# GQA + PSG relation-flip negative design

This note is for building a BLINK-Spatial-style negative QA set from GQA and
PSG scene graphs.  The goal is to use true scene-graph triplets as evidence, but
ask a false spatial question by replacing the true relation with an incompatible
relation.

Important rule: never put the false queried relation into `relation_targets`.
`relation_targets` should contain only true triplets from GQA/PSG, plus optional
extra true triplets involving the mentioned objects.

## Source inventories

### PSG predicate classes

PSG has 56 predicates:

`over`, `in front of`, `beside`, `on`, `in`, `attached to`, `hanging from`,
`on back of`, `falling off`, `going down`, `painted on`, `walking on`,
`running on`, `crossing`, `standing on`, `lying on`, `sitting on`,
`flying over`, `jumping over`, `jumping from`, `wearing`, `holding`,
`carrying`, `looking at`, `guiding`, `kissing`, `eating`, `drinking`,
`feeding`, `biting`, `catching`, `picking`, `playing with`, `chasing`,
`climbing`, `cleaning`, `playing`, `touching`, `pushing`, `pulling`,
`opening`, `cooking`, `talking to`, `throwing`, `slicing`, `driving`,
`riding`, `parked on`, `driving on`, `about to hit`, `kicking`, `swinging`,
`entering`, `exiting`, `enclosing`, `leaning on`.

### GQA high-frequency relation families

GQA has 310 unique relation strings.  The useful families for this task are:

- Horizontal: `to the left of`, `to the right of`.
- Vertical: `above`, `below`, `under`, `underneath`, `beneath`, plus
  `higher than`.
- Support/surface: `on`, `on top of`, `sitting on`, `standing on`,
  `lying on`, `walking on`, `running on`, `riding on`, `parked on`,
  `driving on`, `painted on`, `mounted on`, `perched on`, `resting on`,
  `skating on`, `skiing on`, `balanced on`, `displayed on`, etc.
- Depth: `behind`, `in front of`, `standing behind`,
  `standing in front of`, `sitting behind`, `sitting in front of`,
  `walking behind`, `parked behind`, `parked in front of`.
- Containment: `in`, `inside`, `standing in`, `sitting in`, `lying in`,
  `walking in`, `flying in`, `swimming in`, `wading in`, `kept in`,
  `stuck in`, `displayed in`, `cooked in`, `contain`, `enclosing`.
- Proximity: `near`, `next to`, `beside`, `by`, `with`, `close to`,
  `standing near`, `standing next to`, `standing by`, `sitting near`,
  `sitting next to`, `sitting beside`, `parked near`, `parked next to`,
  `parked beside`, `walking near`, `walking next to`, `growing near`.
- Contact-ish: strict `touching`, plus looser evidence-only relations such as
  `attached to`, `connected to`, `leaning on`, `leaning against`,
  `stuck on`, `plugged into`, `tied to`, `chained to`, `mounted to`.
- Gaze/facing: `facing`, `looking at`, `watching`, `staring at`,
  `observing`, `looking toward`.
- Cover/surround: `covered by`, `covered in`, `covered with`, `covering`,
  `surrounded by`, `surrounding`, `full of`, `filled with`.
- Relative size: `taller than`, `larger than`, `smaller than`,
  `longer than`, `shorter than`, `bigger than`.

Many GQA relations are not safe as primary flip sources: `of`, `at`, `with`,
`by`, `on the side of`, `on the front of`, `on the back of`, `on the edge of`,
`holding`, `wearing`, `eating`, `drinking`, `playing`, `making`, `using`,
and most action verbs.  These are useful as extra evidence triplets, but should
not usually be used as the direct relation to flip.

## Safe inversion rules

The table below gives the main rule set.  "False query variants" should be
sampled diversely, not deterministically.  For example, `above` can be flipped
to `under`, `beneath`, or `below`.

| True relation family | GQA raw examples | PSG raw examples | False query variants | Validation |
|---|---|---|---|---|
| left-of | `to the left of` | derived from bbox only | `to the right of`, `at the right side of`, `right of` | require subject center clearly left of object; query right must be false by bbox |
| right-of | `to the right of` | derived from bbox only | `to the left of`, `at the left side of`, `left of` | require subject center clearly right of object |
| above / over | `above`, `flying above`, `hanging above`, `over`, `flying over`, `jumping over`, `draped over` | `over`, `flying over`, `jumping over` | `under`, `beneath`, `below` | require subject center above object or strong raw relation; false below must fail by bbox |
| below / under | `below`, `under`, `underneath`, `beneath`, `standing under`, `sitting under`, `hanging from` | `hanging from` when bbox supports below | `above`, `over`, `on top of` | require subject center below object; avoid `hanging from` unless bbox confirms |
| on / support | `on`, `on top of`, `standing on`, `sitting on`, `lying on`, `walking on`, `running on`, `riding on`, `parked on`, `driving on`, `painted on`, `mounted on`, `perched on`, `resting on`, `skating on`, `skiing on` | `on`, `standing on`, `sitting on`, `lying on`, `walking on`, `running on`, `painted on`, `parked on`, `driving on` | `under`, `beneath`, `below`, `inside` | require true object is not below/inside subject by bbox |
| behind | `behind`, `standing behind`, `sitting behind`, `walking behind`, `parked behind`, `growing behind` | none direct, can be derived if PSG bbox/depth not available only use labels from caption? generally avoid PSG-derived behind | `in front of`, `at the front of` | use raw label; bbox cannot fully verify depth, so keep moderate quota |
| in-front-of | `in front of`, `standing in front of`, `sitting in front of`, `parked in front of` | `in front of` | `behind`, `at the back of` | use raw label; keep moderate quota |
| near / next-to / beside | `near`, `next to`, `beside`, `close to`, `standing near`, `standing next to`, `sitting near`, `parked near`, `parked next to`, `walking next to` | `beside` | `far from`, `far away from`, `away from` | require bbox distance not large; do not use if boxes are ambiguous tiny objects |
| far geometry | derived from distant object pairs | derived from distant object pairs | `near`, `next to`, `beside`, `touching` | must be bbox-derived; object boxes far apart; no explicit near/contact relation between the pair |
| strict touching | `touching` only | `touching` | `far from`, `far away from`, `away from` | use strict raw `touching` as positive evidence; do not treat `attached to` as `touching=yes` |
| non-touching geometry | pairs with clear bbox gap | pairs with clear bbox gap | `touching` | require edge gap > threshold; best if a different true spatial relation exists between the pair |
| inside / in | `in`, `inside`, `standing in`, `sitting in`, `lying in`, `walking in`, `flying in`, `swimming in`, `wading in`, `kept in`, `stuck in`, `displayed in` | `in` | `outside of`, `containing`, `contain` | require subject bbox mostly inside object bbox when possible; `outside of` should be false |
| contain / enclose | `contain`, `enclosing`, `surrounding` | `enclosing` | `inside`, `in`, `outside of` | require object bbox mostly inside subject bbox when possible |
| attached/contact-ish | `attached to`, `connected to`, `leaning on`, `leaning against`, `stuck on`, `plugged into`, `tied to`, `chained to`, `mounted to` | `attached to`, `leaning on` | `far from`, `far away from`, `away from` | use as evidence for not-far only; do not use as positive `touching` |
| facing / looking-at | `facing`, `looking at`, `watching`, `staring at`, `observing`, `looking toward` | `looking at` | `facing away from`, `looking away from` | strictest source is `facing`; `looking at` is acceptable but noisier |
| relative size | `taller than`, `larger than`, `bigger than`, `longer than`, `shorter than`, `smaller than` | none | inverse size relation | low quota; not BLINK-Spatial core but useful for hard binary relations |

## Diverse false-query pools

Use several aliases for each queried false relation, because BLINK uses varied
surface forms.

- `above` truth -> sample false from: `under`, `beneath`, `below`.
- `below/under/beneath` truth -> sample false from: `above`, `over`,
  `on top of`.
- `left` truth -> sample false from: `right of`, `to the right of`,
  `at the right side of`.
- `right` truth -> sample false from: `left of`, `to the left of`,
  `at the left side of`.
- `behind` truth -> sample false from: `in front of`, `at the front of`.
- `in front of` truth -> sample false from: `behind`, `at the back of`.
- `on/on top of` truth -> sample false from: `under`, `beneath`, `below`,
  occasionally `inside` if bbox confirms.
- `in/inside` truth -> sample false from: `outside of`, `containing`,
  `contain`.
- `contain/enclosing` truth -> sample false from: `inside`, `in`,
  `outside of`.
- `near/next to/beside/touching/contact-ish` truth -> sample false from:
  `far from`, `far away from`, `away from`.
- distant bbox pairs -> sample false from: `near`, `next to`, `beside`,
  `touching`.
- `facing/looking at` truth -> sample false from: `facing away from`,
  `looking away from`.

## Relations to avoid as primary flip sources

These can still be extra true triplets, but should not usually define the false
QA relation:

- Generic/prepositional: `of`, `at`, `with`, `by`, `between`,
  `on the side of`, `on the front of`, `on the back of`,
  `on the bottom of`, `on the edge of`, `on the other side of`.
- Attribute-ish or part-whole: `full of`, `filled with`, `part of`-like
  labels, `larger/smaller` if not explicitly desired.
- Action-only: `holding`, `wearing`, `carrying`, `using`, `eating`,
  `drinking`, `feeding`, `playing`, `making`, `reading`, `opening`,
  `cooking`, `throwing`, `slicing`, `cleaning`, `talking to`, etc.
- Motion/path when direction is unclear: `crossing`, `going down`,
  `jumping from`, `falling off`, `entering`, `exiting`, `coming from`,
  `leaving`, `approaching`, `following`, `walking toward`, etc.
- Cover/overlap unless bbox validates the queried false relation:
  `covered by`, `covered in`, `covered with`, `covering`, `wrapped in`,
  `wrapped around`.

## Proposed sampling strategy

For a 10k relation-flip negative set:

- Horizontal left/right: 1500
- Vertical above/below/under/beneath/over: 1800
- On/support: 1500
- Depth behind/in-front-of: 1200
- Containment inside/in/contain/enclosing: 1200
- Proximity/far negatives: 1200
- Touching/non-touching: 800
- Facing/looking-at/facing-away: 500
- Low-quota size or other verified hard relations: 300

Within each family:

1. Sample both GQA and PSG when both are available.
2. Use log-count or capped sampling so `to the left of`, `to the right of`,
   and `on` do not dominate.
3. Give each raw relation variant a small floor when enough examples exist.
   For example, within support, sample from `standing on`, `sitting on`,
   `lying on`, `walking on`, `parked on`, etc., not only raw `on`.
4. For every chosen example, randomly choose one false surface form from that
   family's false-query pool.

## Required validation

Each generated QA should pass:

1. The subject and object names are unique enough in the image.  If the same
   noun appears multiple times, either skip or add a safe attribute from the
   scene graph.
2. The true relation triplet is present in `relation_targets`.
3. The false queried relation is not present between the same object pair.
4. When the false relation is geometric (`left/right/above/below/near/far/
   touching/inside/contain`), bbox validation confirms it is false.
5. Add 0-2 extra true triplets involving the subject or object when available.
   These should be evidence/distractors, not false facts.
6. If no reliable bbox or object identity is available, skip the sample.

## QA template

Use BLINK-like binary phrasing:

`Question: Is the {subject} {false_relation_phrase} the {object}?`

Choices:

`A. yes`

`B. no`

Answer:

`B`

Examples:

- True triplet: `cat -- to the left of -- dog`.
  Question: `Is the cat to the right of the dog?`
- True triplet: `person -- standing on -- skateboard`.
  Question: `Is the person under the skateboard?`
- True triplet: `cup -- inside -- cabinet`.
  Question: `Does the cup contain the cabinet?`
- True triplet: `person -- looking at -- dog`.
  Question: `Is the person facing away from the dog?`
- Bbox-derived true relation: `bus -- far from -- bicycle`.
  Question: `Is the bus touching the bicycle?`

## Expected benefit

The current PSG false set teaches a few negative types very strongly, especially
`under`, `behind`, and `outside of`.  The missing piece for BLINK Spatial is a
broader set of hard false relations where both objects exist but the queried
relation is wrong.  This design should especially improve false/no questions
for `touching`, `left/right`, `facing`, `near/far`, `above/below`, and
`inside/contain`.
