# MechFaber

**Design real machines from your coding agent.** You own the topology and the
geometry; deterministic solvers own every number — reach, duty, packing,
statics, simulation, and which real components exist at what rating.

Ask for a robot arm, get a buildable machine with a bill of materials that
cites its sources.

---

## Install

### Claude Code

```
/plugin marketplace add Argentron-Technologies/mechfaber-agent
/plugin install mechfaber
```

That is the whole setup. **The plugin carries both halves**: the `mechfaber`
skill, which teaches the design loop, and the MCP server that runs the CAD,
the physics and the firmware. A skill on its own would give you the
instructions and nothing to execute them with.

### Codex, OpenCode, Crush, and others

Agent Skills is an open standard, so **the same `SKILL.md` works unmodified**
across thirty-odd agents. Only two things differ: where your agent looks for
skills, and how it declares an MCP server.

Copy `plugins/mechfaber/skills/mechfaber/` into the skills folder, then add
the server:

| Agent | Skills folder | MCP server |
|---|---|---|
| **Codex** | `~/.agents/skills/` | `~/.codex/config.toml` |
| **OpenCode** | `~/.agents/skills/` or `~/.claude/skills/` | `opencode.json` |
| **Crush** | `~/.config/crush/skills/` <br>(`%LOCALAPPDATA%\crush\skills\` on Windows) | `crush.json` |

<details>
<summary>Server config for each</summary>

**Codex** — `~/.codex/config.toml`
```toml
[mcp_servers.mechfaber]
url = "https://api.mechfaber.com/mcp"
```

**OpenCode** — `opencode.json`
```json
{
  "mcp": {
    "mechfaber": {
      "type": "remote",
      "url": "https://api.mechfaber.com/mcp",
      "enabled": true
    }
  }
}
```

**Crush** — `crush.json`
```json
{
  "mcp": {
    "mechfaber": {
      "type": "http",
      "url": "https://api.mechfaber.com/mcp"
    }
  }
}
```
</details>

`~/.agents/skills/` is the nearest thing to a universal location — Codex and
OpenCode both read it, and Crush reads `~/.config/agents/skills/`. Claude Code
and OpenCode also read `~/.claude/skills/`. **Crush does not read
`.claude/skills`**, so it needs its own folder.

### Signing in

You need a MechFaber account. The server speaks OAuth, so your agent opens a
browser on first use — **there is no API key to paste, and nothing secret in
this repository.**

Self-hosting or working locally? Point it elsewhere:

```
export MECHFABER_MCP_URL=https://localhost:44301/mcp
```

---

## What you get

Every claim is gated by something that can fail:

- **CAD** — real B-reps through build123d/OCCT, assembled with *joints* rather
  than transforms, so the physics model comes out of the geometry instead of
  being typed a second time and drifting from it.
- **Geometry gates** — exact interference volume between every unjointed pair
  of solids; a travel sweep that poses each revolute through its declared
  range; bolt lines measured off the parts, because a mate in the model is not
  a screw in the metal.
- **Physics** — closed-form statics checked against a stepped MuJoCo
  simulation. When the two disagree, one of them is wrong, and that is worth
  knowing before you buy anything.
- **Parts** — actuator ratings transcribed from a cited datasheet page, and
  the vendor STEP measured rather than guessed at. A rating with no citation
  is a number somebody typed.
- **Firmware** — your control loop compiled and run on an emulated MCU against
  the machine, its loom and its pack together. Torque draws current, current
  sags the pack, a sagged rail caps what a joint can deliver, and an MCU below
  its brownout threshold stops your loop mid-run.

The design loop itself is in
[`SKILL.md`](plugins/mechfaber/skills/mechfaber/SKILL.md) — worth reading even
if you never install it, since it is mostly a list of the ways machine design
goes quietly wrong.

---

## Layout

```
.claude-plugin/marketplace.json     this repo, as a Claude Code marketplace
plugins/mechfaber/
  .claude-plugin/plugin.json        plugin metadata
  .mcp.json                         the MCP server, registered on install
  skills/mechfaber/SKILL.md         the design loop — the whole skill
```

## Links

- [mechfaber.com](https://mechfaber.com) — what it is
- [app.mechfaber.com](https://app.mechfaber.com) — the Studio: viewer,
  simulation, wiring, bill of materials

## Licence

[LGPL-3.0](COPYING.LESSER). Weak copyleft, chosen deliberately: drop the skill into a
proprietary project and that project stays yours, but changes to the skill
itself stay open.

LGPL-3.0 is defined as GPL-3.0 plus a set of additional permissions.
[`COPYING.LESSER`](COPYING.LESSER) carries the permissions; the GPL they
build on is at [gnu.org/licenses/gpl-3.0.txt](https://www.gnu.org/licenses/gpl-3.0.txt)
rather than copied in here.
