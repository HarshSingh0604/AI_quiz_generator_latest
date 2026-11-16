# CLAUDE.md

Notes for Claude (in VS Code or the Claude app) working on this repo. Read this first; README.md has the user-facing setup and full API list.

## What this is
Spring Boot 3.2 / Java 17 web app: teachers upload study material (TXT, PDF, DOCX, JPG/PNG), an LLM turns it into multiple-choice quizzes, students take them, and a leaderboard ranks scores. If the AI call fails, a rule-based generator makes the quiz instead. It's a college capstone / portfolio project, so keep changes simple and easy to explain.

## Run and build
- Always run from the project root: `spring-dotenv` reads `.env` from the working directory.
- Dev: `mvn spring-boot:run`
- Build: `mvn clean package`, then `java -jar target/ai-quiz-generator-1.0.0.jar`
- App runs at http://localhost:8080 and needs MySQL running (`brew services start mysql`).
- You can run without MySQL using the in-memory H2 flags shown in README ("Quick demo without MySQL").
- There are no automated tests yet (`src/test` doesn't exist). Verify changes by building and hitting the endpoints, e.g.
  `curl -F file=@sample-material.txt http://localhost:8080/api/materials/upload` then
  `curl -X POST "http://localhost:8080/api/quiz/generate/1?questionCount=5"`.
- Not a git repo at the moment.

## Secrets
- `.env` holds `GROQ_API_KEY`, `DB_USERNAME`, `DB_PASSWORD` (optional `OCR_SPACE_KEY`). Never print, commit, or hardcode its values. Never put keys in `application.properties`.
- `start.bat` sets DB credentials itself and overrides `.env`.

## Code layout (`src/main/java/com/quizgen`)
- `controller/`: REST endpoints under `/api` (auth, materials, quiz, attempts, leaderboard).
- `service/QuizGenerationService`: entry point for generation. Tries AI, falls back to rule-based when `app.quiz.fallback.enabled=true`, saves the `Quiz`, then its `Question`s. Clamps question count to 1–20. Records the method used as `AI` or `RULE_BASED`.
- `service/AIQuizGenerationService`: OpenAI-compatible client (`com.theokanning.openai-gpt3-java`) pointed at Groq through a custom Retrofit base URL, plus an OkHttp interceptor that rewrites `/v1/` → `/openai/v1/` for api.groq.com. Splits long text into overlapping chunks, runs them on a small thread pool, then merges and de-duplicates. Retries are a hand-written loop (`generateQuestionsWithRetry`), not `@Retryable`. Results are `@Cacheable("quizzes")` by model + count + text hash.
- `service/RuleBasedQuizGenerationService`: fallback that picks sentences from the text and uses generic wrong answers.
- `service/TextExtractionService`: PDFBox / Apache POI locally. Scanned PDFs and images go to the OCR.space API.
- `model/`: JPA entities with plain getters/setters (Lombok is in the pom but unused). Relations are stored as raw ID columns (e.g. `Question.quizId`), not `@ManyToOne`.
- `src/main/resources/static/`: plain HTML/CSS/vanilla JS frontend, no build step. `js/auth.js` keeps the logged-in user in `sessionStorage`.

## Things to know before changing code
- Schema comes from the entities (`spring.jpa.hibernate.ddl-auto=update`). `schema.sql` is NOT run (`spring.sql.init.mode=never`), so if you change an entity, update `schema.sql` too to keep them in step.
- `QuizController` rejects material over 3,000 words. That's separate from the 3,000-character chunk size.
- Groq retires models. If generation silently falls back to rule-based, check the logs for `AI generation failed` and look at `openai.model` in `application.properties`.
- Auth and roles are enforced only in the browser. The API itself is open and passwords are unsalted SHA-256. This is a known limitation (README lists it). Don't claim it's secure. Spring Security + BCrypt + JWT is the planned fix.
- Rebuild after editing `application.properties`.

## Conventions
- Constructor/field injection with `@Autowired` and SLF4J loggers match the existing style. Keep the controller → service → repository layering.
- Keep the frontend dependency-free (no npm, no framework).
- When a behaviour or config change affects users, update README.md too.
