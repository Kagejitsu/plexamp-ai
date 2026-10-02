# plexamp-ai

Run Plexamp's **Sonic Sage** AI playlists on a **local LLM** instead of OpenAI,
on Linux.

Plexamp hardcodes `https://api.openai.com` for Sonic Sage. `plexamp-ai`
builds a patched copy of Plexamp in your home directory that points Sonic Sage
at any OpenAI-compatible server instead:
[Lemonade](https://lemonade-server.ai), [Ollama](https://ollama.com),
`llama-server` from llama.cpp, LM Studio, vLLM, and so on. You choose the model.
Your playlist prompts never leave your machine, and you need no OpenAI account.

The installed Plexamp package is never modified, and the regular Plexamp stays
available next to the patched one.

> Not affiliated with or endorsed by Plex, Inc. This repository contains no
> Plex code: it patches the copy of Plexamp that is already on your system.

## Requirements

- Arch Linux or a derivative, with Plexamp installed as an AppImage at
  `/usr/bin/Plexamp.AppImage`. Both AUR packages
  [`plexamp-appimage`](https://aur.archlinux.org/packages/plexamp-appimage) and
  [`plexamp-beta-appimage`](https://aur.archlinux.org/packages/plexamp-beta-appimage)
  install it there. Other locations work with `PLEXAMP_APPIMAGE=/path/to/Plexamp.AppImage`.
- Python 3, with the standard library only.
- An OpenAI-compatible LLM server.

`plexamp-bin`, which runs Plexamp on the system Electron, is not supported yet.

## Install

From the AUR:

```sh
paru -S plexamp-ai     # or yay, or makepkg -si from the AUR repo
```

By hand:

```sh
install -Dm755 plexamp-ai plexamp-ai-patch -t /usr/bin/
install -Dm644 plexamp-ai.desktop -t /usr/share/applications/
install -Dm644 pacman/plexamp-ai.hook -t /usr/share/libalpm/hooks/
install -Dm755 pacman/plexamp-ai-hook /usr/share/libalpm/scripts/plexamp-ai
```

## Usage

1. Choose your server and model once. Lemonade's default port is used unless you
   say otherwise:

   ```sh
   # Lemonade (the default)
   plexamp-ai-patch --base http://localhost:13305/api/v1 --model Gemma-4-26B-A4B-it-GGUF

   # Ollama
   plexamp-ai-patch --base http://localhost:11434/v1 --model gemma4:26b
   ```

   These settings are remembered. Later runs without `--base` or `--model`
   keep them.

2. Start **Plexamp AI** from your app menu, or run `plexamp-ai`. Quit the regular
   Plexamp first. Electron allows only one instance, so a second launch just
   brings the window that is already open to the front.

3. In Plexamp, open **Settings → Sonic Sage** and put **any text** in the OpenAI
   API key field, for example `local`. Plexamp hides Sonic Sage while the field is
   empty. The key is sent as a bearer token, which local servers ignore.

Check the current settings with `plexamp-ai-patch --status`.

To try another model or server for one session, without re-patching:

```sh
PLEXAMP_AI_MODEL=qwen3:30b-a3b-instruct plexamp-ai
PLEXAMP_AI_BASE=http://my-desktop:11434/v1 plexamp-ai
```

### Choosing a model

Sonic Sage sends the playlist title together with a "you are a talented DJ"
system prompt. It expects back one JSON object per line, in the form
`{"artist": ..., "track": ..., "why": ...}`, and then looks each track up in
your library. What matters most:

- **Music knowledge.** The model has to name songs that really exist. Made-up
  tracks are simply skipped, so a model that knows little gives short playlists.
  Larger models know more.
- **No reasoning or "thinking" models.** Plexamp only reads
  `choices[0].delta.content`. A thinking model such as gpt-oss, DeepSeek-R1, or
  Qwen3 with thinking on stays silent while it thinks, or breaks the JSON.
  Use an instruct model, or turn thinking off.
- **Speed.** The answer streams in, so a fast MoE model feels much quicker.

Gemma 4 26B-A4B-it is a good default. It is a fast MoE with broad knowledge,
it does not think before answering, and it copes well with titles in other
languages. Do not expect a local model to know obscure deep cuts the way
GPT-4o does.

## Updates and the pacman hook

The patched copy lives in `~/.local/share/plexamp-ai/app`, about 300 MB, and
is rebuilt automatically when it goes stale:

- **At launch.** `plexamp-ai` re-patches if Plexamp or plexamp-ai has changed
  since the last patch.
- **After pacman transactions.** A hook re-patches the copy of every user who has
  one, as soon as Plexamp or plexamp-ai is upgraded. The patcher runs as that
  user, never as root.

The patcher checks that every piece of code it replaces appears exactly as
often as expected. If a Plexamp update changes that code, the patch **fails
loudly instead of half-patching**. In pacman this shows as a yellow `WARNING`
and never fails the transaction. Your previous patched copy is left in place,
so it keeps working, but it stays on the older Plexamp version until
plexamp-ai is updated. Please open an issue with the Plexamp version when this
happens.

## How it works

Plexamp is an Electron app. All of its UI code is in a single bundle, `index.js`,
inside `resources/app.asar`. The patcher:

1. runs `Plexamp.AppImage --appimage-extract` into a temporary directory,
2. replaces `https://api.openai.com/v1/models` and `.../v1/chat/completions` with
   the configured base URL,
3. replaces the model names Plexamp chooses (`gpt-4o`, `gpt-4-turbo`, `gpt-4`,
   `gpt-3.5-turbo`) with the configured model,
4. writes the asar back, shifting file offsets and updating the integrity
   hashes, using its own small asar writer. Node and `@electron/asar` are not
   needed.

Both replacements read `process.env.PLEXAMP_AI_BASE` and `PLEXAMP_AI_MODEL`
first, which is why you can override them at launch. Plexamp leaves Electron's
asar-integrity fuse off, so the edited archive loads normally.

## Limitations

- **AI artwork** (DALL-E) still points at OpenAI. Local servers do not return
  image URLs the way Plexamp expects, so the artwork button will show an error.
- Only tested on Plexamp **4.13.2**. Other versions, including the beta, are
  accepted only if the patch applies cleanly. Otherwise you get the loud failure
  described above.
- Linux only.

## Uninstall

```sh
paru -R plexamp-ai
rm -rf ~/.local/share/plexamp-ai    # for each user
```

## License

MIT. See [LICENSE](LICENSE).
