<div align="center">
<img height="180" alt="Poku's Logo" src="https://raw.githubusercontent.com/wellwelwel/poku/main/.github/assets/readme/poku.svg">

# @pokujs/skills

Enjoying **Poku**? [Give him a star to show your support](https://github.com/wellwelwel/poku) 🌟

---

📘 [**Documentation**](./SKILL.md)

</div>

---

🧠 [**@pokujs/skills**](https://github.com/pokujs/skills) is a **Poku** skill repository for agents and contributors working with Poku tests.

> [!TIP]
>
> Use this repository when you need accurate guidance for writing tests, explaining Poku's execution model, or debugging assertions, configuration, and integration setups.

---

## Quickstart

### Add the Skill

```bash
npx skills add pokujs/skills
```

### Start with the Main Guide

- [SKILL.md](./SKILL.md): entry point, overview, and quick reference
- [assertions.md](./assertions.md): assertion APIs and examples
- [background-services.md](./background-services.md): processes, ports, and readiness helpers
- [configuration.md](./configuration.md): configuration options, reporters, and CLI flags
- [patterns.md](./patterns.md): unit, integration, and end-to-end testing patterns

### Use It For

- writing or debugging tests with Poku
- explaining how test files execute and isolate state
- checking assertion APIs and runtime-specific behavior
- working with background services in integration or e2e tests
- looking up configuration and CLI options

---

## How It Is Organized

- `SKILL.md` is the main entry point and should stay concise
- supporting files hold the detailed reference material that would otherwise make the skill too large
- the repository is intended to be read incrementally: overview first, then the focused reference file for the task at hand

---

## Maintenance

Keep `SKILL.md` aligned with the supporting reference files so the overview and deeper reference material do not drift apart.

---

## License

[MIT](./LICENSE)