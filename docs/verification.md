# Verification

The point of this page is that "I edited a config file" is not evidence that a plugin loads. These are the checks that actually prove it, in increasing order of strength.

## 1. YAML parses

`cordis.patch.yml` supports `!!js` tags, so a plain YAML parser is what you want for a syntax check — not a loader that tries to evaluate the tags.

```bash
npx js-yaml ~/.dsh/profiles/<profile>/cordis.patch.yml > /dev/null && echo "YAML OK"
```

Then confirm the structure, not just that it parsed:

```bash
npx js-yaml cordis.patch.yml | node -e '
let s="";process.stdin.on("data",d=>s+=d).on("end",()=>{
  const d=JSON.parse(s);
  const p=d.find(e=>e.id==="llm-pi-ai");
  for(const [k,v] of Object.entries(p.config.providers))
    console.log(k,"| api=",v.api,"| key=",v.apiKeyEnv);
});'
```

Every `key=` value must be a **variable name**. If you see a literal secret here, stop and move it to the credential store.

## 2. The profile composes

`--dump-config` applies every bundle layer plus your patch and prints the composed tree without mounting anything. It is the cheapest real check.

```bash
dsh --profile <profile> --dump-config > /tmp/composed.yml
echo "exit=$?"          # must be 0
```

## 3. The plugin actually appears

Dumping without error only tells you the patch applied. Look for the entries DSB inserts:

```bash
grep -nE "botharness|preset-deepseekbot" /tmp/composed.yml
```

Expected: `botharness-core`, `botharness-client`, `botharness-browser`, `botharness-computer`, and a `preset-deepseekbot` agent preset. Missing entries mean `deepseekbot` is not in `dsh.profile.bundles`.

## 4. Routes survive composition

```bash
awk '/id: llm-pi-ai/,/id: agent-default-model/' /tmp/composed.yml
```

This is where a duplicate `llm-pi-ai` entry bites: the later map replaces the earlier one, so routes you added first quietly disappear. If a route is missing here, you have two entries.

## 5. Boot it for real

Run an **isolated** instance so a broken plugin cannot take down your working app. Note the argument order — the profile is a global flag and the app is positional:

```bash
DSH_HOME=/tmp/dsh_verify cp -R ~/.dsh/profiles /tmp/dsh_verify/
DSH_HOME=/tmp/dsh_verify dsh --profile <profile> --host 127.0.0.1 --port 61999 --no-open
```

A healthy boot prints:

```
dsh web: http://127.0.0.1:61999/?token=...
```

Without the `?token=` the page returns 401, which looks like a failure but is the browser-trust fence doing its job.

> A profile named `desktop` is owned exclusively by the Electron app. Copy it to a new name to boot it from the CLI.

## 6. Confirm the UI mounted

With the tokenised URL open, the DOM should contain `Bot mode`, `bot-mode`, and injected `@botharness/*` styles. Their absence means the plugin loaded on the host but its client bundle did not.

## Cleanup

```bash
rm -rf /tmp/dsh_verify /tmp/composed.yml
```
