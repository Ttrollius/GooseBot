# GooseBot
# Municipal Website Chatbot: Tock Option

Oct 28, 2026 · Adam & Evan

## Summary

Tock can meet the no-LLM requirement, but only if it is deployed without its Gen AI add-on. It is an actively maintained, open source chatbot platform that understands typed questions using small language-understanding models trained only on the town's own examples, all hosted on town servers.

Tock ("The Open Conversation Kit") was built by the French rail operator SNCF for its own customer chatbot and released as open source. It sits between the earlier options and a full AI assistant:

|  | Decision tree | FAQ search | Tock (no Gen AI) |
| --- | --- | --- | --- |
| How residents use it | Click menus | Type a question | Type a question, have a back-and-forth conversation |
| How it understands text | It doesn't | Word overlap scoring | Trained intent classifier + entity extraction (dates, places, numbers) |
| Where answers come from | Staff-written | Staff-written | Staff-written, or live lookups from town systems |
| Can it give an unapproved answer? | No | No | No, with Gen AI off |
| Multi-step conversations | Menu paths only | No | Yes (e.g. asks for an address, then checks a schedule) |
| Staff admin screen | Build one, or edit files | Build one, or edit files | Included (Tock Studio) |
| Effort to set up and run | Low | Low to medium | Medium to high: several services, a database, developers who know Kotlin or its APIs |

Tock is worth it if the town wants conversations, not just answers: booking, reporting, checking a status. For answering general questions, the FAQ search option gets most of the value with far less to run.

## How Tock works

**In plain terms:** staff teach Tock what residents mean by giving it example sentences. "When's garbage day", "what day is trash pickup" and "bins collected when" are all labelled as the same request. Tock learns to recognize that request in new wordings, then follows a conversation script (a "story") that staff wrote for it. It never writes its own replies.

### Example conversation

1. Resident: "when do they collect garbage on my street?"
2. Tock recognizes the request *garbage schedule*, but needs an address. Bot: "Sure. What's your street name?"
3. Resident: "Water Street"
4. Tock picks out the street name and looks up the schedule from the town's collection data. Bot: "Garbage on Water Street is collected on Tuesdays. Next pickup: Tuesday, November 3."
5. Resident: "what about recycling"
6. Tock keeps the street from earlier and answers for recycling.

That back-and-forth, remembering the street and switching topics, is what Tock adds over the earlier options.

### The main parts

- **Tock Studio:** a web admin screen where staff add example sentences, write answers and FAQs, build stories, and review real conversations and what the bot got wrong.
- **NLU engine (natural language understanding):** classifies each message into an intent and extracts entities. It uses established open source language libraries such as Apache OpenNLP or Stanford CoreNLP, trained only on the town's examples.
- **Duckling:** an open source rules-based component that recognizes dates, times and numbers ("next Tuesday", "3 bags").
- **Bot engine and stories:** decides what happens next. Simple stories are built in Studio without code; complex ones (calling a town system) are written in Kotlin, Python, Node.js, or over a REST API.
- **Database:** MongoDB stores the bot's configuration, training sentences and conversation history.
- **Web connector and chat widget:** a ready-made chat window (Tock React Kit) for the website, talking to the bot through a REST API.

&#91;embedded content: Tock deployment · core services only\]

Everything runs on town-controlled servers. The Gen AI orchestrator is never installed, and outbound internet from these servers is blocked.

## The LLM question: keeping Gen AI off

Tock's core does not use an LLM, but Tock now ships an optional Gen AI add-on, and most recent development is on that add-on. The town's deployment must leave it out entirely.

- **What the add-on is:** a separate Python service (the "Gen AI Orchestrator") that connects Tock to an LLM to generate answers from documents, a technique called RAG. Recent releases have mostly improved this RAG system and its evaluation tools ([26.3.0 notes](https://github.com/theopenconversationkit/tock/releases/tag/tock-26.3.0)), and the September 2026 release added Amazon Bedrock as an LLM provider ([26.9.0 notes](https://github.com/theopenconversationkit/tock/releases/tag/tock-26.9.0)).
- **It is a separate deployment:** Tock's official Docker setup provides the Gen AI orchestrator as its own compose file, apart from the main stack ([tock-docker](https://github.com/theopenconversationkit/tock-docker)). Not deploying it means no component exists that could talk to an LLM.
- **How to prove it to the client:**
  - Deploy only the core services (Studio, NLU, bot API, Duckling, database). No Gen AI orchestrator container, no vector database, no LLM API keys anywhere.
  - Block outbound internet from the bot servers at the firewall, allowing only the town's own systems.
  - Confirm in Tock Studio that the Gen AI / RAG settings are empty and unused, and check this after every upgrade.
  - Write the requirement into the project's acceptance criteria: "no component sends resident or town data to a language model."
- **Wording for the client:** "Tock core, self-hosted, NLU-based, Gen AI module not installed." As with Rasa, anyone who searches for Tock will now find a lot of LLM material, so be specific.

## Project health and support

Unlike Rasa Open Source, Tock is actively maintained, with regular releases through 2026. The main risk is the opposite one: the maintainers' attention is on Gen AI, not the classic NLU features the town would use.

| Check | Finding |
| --- | --- |
| Licence | Apache 2.0, free for commercial and public-sector use ([IDB listing](https://knowledge.iadb.org/en/open-knowledge/code-development/open-source-solution/tock-chatbot)) |
| Latest release | 26.9.0, September 29, 2026 ([release notes](https://github.com/theopenconversationkit/tock/releases/tag/tock-26.9.0)) |
| Release rhythm | Major versions in March and September, with patch releases between ([releases](https://github.com/theopenconversationkit/tock/releases)) |
| Home | Moved from SNCF's GitHub to its own organization in November 2024 ([old repo](https://github.com/voyages-sncf-technologies/tock)) |
| Main language | Kotlin, with TypeScript for Studio and Python for the Gen AI parts ([repo](https://github.com/Pazuzzu/tock)) |
| Community | Small and largely French; support is through GitHub issues and a Gitter chat, with no commercial support vendor |

Points to raise with the client:

- No company sells support for Tock, so the town or its contractor owns troubleshooting and upgrades.
- The team needs someone comfortable with Kotlin/JVM services, Docker and MongoDB.
- Some optional NLU libraries carry different licences from Tock itself (Stanford CoreNLP is GPL-licensed). Check the licence of each component actually deployed.

## Strengths, limitations and technical notes

### Strengths

- Understands varied wording and holds multi-step conversations, which the earlier options cannot.
- Every reply is still written or approved by staff; with Gen AI off, nothing is generated.
- Tock Studio gives non-developers a place to add training sentences, edit answers and review what residents asked.
- Built to run on your own servers, with data and models staying in-house.
- Can connect to town systems (collection schedules, permit status) for live answers.
- Proven at scale: it runs SNCF's public-facing customer bot.

### Limitations

- Much more to run than a static widget: several services, a MongoDB replica set, and upgrades twice a year.
- The model is only as good as its training sentences; expect an ongoing task of reviewing missed questions and adding examples.
- Smaller, mostly French-language community; fewer tutorials and answers online than for mainstream tools.
- Development focus is shifting toward Gen AI, so classic NLU features may see fewer improvements.

### Technical notes for implementers

- **Deployment:** the official Docker Compose setup runs MongoDB as a three-node replica set plus separate containers for Studio (admin), the NLP API, the bot API, Duckling and a build worker ([compose example](https://github.com/CreditMutuelArkea/tock-docker/blob/master/docker-compose-reverseproxy.yml)). Start from it, minus the Gen AI compose file.
- **Bot development modes:** stories can be built in Studio (no code) or as code through the Bot API, with clients for Kotlin, JavaScript/Node.js and Python in webhook or WebSocket mode ([API docs](https://doc.tock.ai/tock/master/dev/api.html)).
- **APIs:** a web connector REST API for the chat, an NLU API for querying the model directly, and an admin API for Studio ([API docs](https://doc.tock.ai/tock/master/dev/api.html)). The NLU API is useful for automated tests: send a list of test sentences and check the intents returned.
- **Training data:** start with 15 to 30 example sentences per intent, drawn from real resident emails and call notes. Hold some back as a test set to measure accuracy before launch.
- **Confidence handling:** configure a fallback story for low-confidence matches that offers top suggestions or hands off to staff, the same principle as the FAQ option's threshold.
- **Prototype first:** a one-week proof of concept with 10 intents and one live lookup will show whether the team can run it comfortably.

## Privacy, hosting and website integration

Tock stores every conversation by default so staff can review and improve the bot, which makes data handling more important here than in the simpler options.

### Privacy and hosting

- **Conversation history:** set a retention period and purge older conversations from MongoDB automatically. Keep only what staff actually use to improve training.
- **Personal details:** residents will type names, addresses and phone numbers. Mask them before storage, and limit who can see conversations in Studio through user roles.
- **Live lookups:** stories that query town systems should only return what is already public (a collection day for a street), unless residents sign in. Anything account-specific belongs behind the town's secure login, not in an anonymous chat.
- **Hosting:** run all services, including MongoDB and backups, on town-controlled infrastructure in Canada. Tock Studio should be reachable only from the town network or VPN, never the public internet.
- **Records and privacy law:** stored conversations are town records and may fall under ATIPPA, 2015. Review with the ATIPP coordinator and complete a privacy impact assessment before launch.

### Adding it to the website

- **Chat widget:** Tock React Kit provides a ready-made chat window that can be added as a script or a React module, connects to the bot API, and supports custom styling ([tock-react-kit](https://github.com/theopenconversationkit/tock-react-kit)). The latest release, 25.9.0, added streaming messages and optional chat history expiry ([releases](https://github.com/theopenconversationkit/tock-react-kit/releases)).
- **Host the script yourself:** the quick-start loads the widget from the public unpkg CDN ([quick-start](https://de.github.com/theopenconversationkit/tock-react-kit)). For a municipal site, download a pinned version and serve it from the town's own domain.
- **On the existing site:** one script tag and an empty container element in the shared template, the same embed approach as the earlier options. The bot API sits behind the town's web server or a reverse proxy over HTTPS.
- **Accessibility:** test the widget against WCAG 2.1 AA (keyboard use, screen readers, contrast) and adjust its theme as needed; don't assume the default meets it.

## Recommendation and next steps

Tock is a credible next step up if the town wants real conversations and live lookups, and unlike Rasa Open Source it is still actively developed. Go ahead only with a proof of concept first, because the running costs, not the software, decide whether it fits.

- [ ] Confirm what "more advanced" means to the client: understanding varied wording, multi-step tasks, or live lookups from town systems
- [ ] Confirm the team can support Kotlin/JVM services, Docker and MongoDB long term, or budget for a contractor
- [ ] Build a proof of concept: core Tock only, 10 intents, one live lookup (e.g. collection day by street), firewall blocking outbound traffic
- [ ] Measure accuracy on held-back resident questions and compare against the FAQ search option on the same questions
- [ ] Review retention, masking and Studio access with the ATIPP coordinator
- [ ] Decide between Tock and the simpler options based on the proof-of-concept results

## Sources

- [Tock releases on GitHub](https://github.com/theopenconversationkit/tock/releases)
- [Tock 26.9.0 release notes](https://github.com/theopenconversationkit/tock/releases/tag/tock-26.9.0)
- [Tock 26.3.0 release notes](https://github.com/theopenconversationkit/tock/releases/tag/tock-26.3.0)
- [Tock API documentation](https://doc.tock.ai/tock/master/dev/api.html)
- [tock-docker repository](https://github.com/theopenconversationkit/tock-docker)
- [Tock Docker Compose example (reverse proxy)](https://github.com/CreditMutuelArkea/tock-docker/blob/master/docker-compose-reverseproxy.yml)
- [Tock React Kit](https://github.com/theopenconversationkit/tock-react-kit) and [its releases](https://github.com/theopenconversationkit/tock-react-kit/releases)
- [Inter-American Development Bank open source listing for Tock](https://knowledge.iadb.org/en/open-knowledge/code-development/open-source-solution/tock-chatbot)
- [Archived SNCF repository (move notice)](https://github.com/voyages-sncf-technologies/tock)
