# Ship it without typing — the whole thing, step by step

Nineteen steps. You do every one of them by hand: a browser, a terminal, and
your own typing. Nothing here needs an assistant, and nothing here is magic.

By the end, changing the colour of the app will take you one `git push` and
about four minutes, and you will not log into the server to do it.

![how it fits together](docs/diagram-4.png)

> **Copy commands from this file on GitHub, not from the PDF handout.** The
> handout's font joins `--` and `//` into one character when you copy it,
> which breaks `--name` and `https://` in ways the error message disguises.

---

## Part A · On your laptop

### 1 · Fork it

Actions only runs in a repo you own. **Fork**, top right of the repo page.

### 2 · Clone it, next to last week's folder

Put it beside your Week 05 folder. Step 3 is simpler if you do.

```bash
cd WHEREVER-YOUR-WEEK-05-FOLDER-LIVES
git clone https://github.com/YOUR-NAME/buildrbank-cicd.git
cd buildrbank-cicd
```

✓ `ls ..` shows both `buildrbank-deploy` and `buildrbank-cicd`.

### 3 · Bring your keys across

```bash
cp ../buildrbank-deploy/.env .env
```

`.env` is in `.gitignore`. It is the one file that never goes into git.

> **If your Week 05 folder is somewhere else,** use the full path — and put
> quotes around it if any folder name contains a space:
>
> ```bash
> cp "/Users/you/My Documents/buildrbank-deploy/.env" .env
> ```
>
> Without the quotes the shell splits the path at every space and you get
> `cp: .env: Not a directory`, which does not mention spaces at all.

✓ `cat .env` shows your OpenRouter, Supabase and Langfuse keys.

### 4 · Prove it still works before you change anything

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install -r requirements.txt -r requirements-dev.txt
python scripts/check.py
```

Windows: `.venv\Scripts\activate`, and use `python`, not `python3`.

✓ Six green ticks.

> **`pip` errors about building wheels?** Your Python is newer than the one
> the container uses. Install Python 3.12 and run
> `python3.12 -m venv .venv` instead.

---

## Part B · One server

### 5 · Launch the instance

EC2 → **Launch instance**.

| | |
|---|---|
| Name | `buildrbank-cicd` |
| Image | Ubuntu, 64-bit **x86** |
| Size | `t3.micro` |
| Key pair | reuse your Week 05 `.pem` if you still have it, otherwise create one — it downloads **once** |
| Firewall | allow **22**, **8000** and **8001**, source **Anywhere-IPv4** |

Three ports, not two. `8000` is production, `8001` is the dev sandbox, and
you will meet both in step 15.

Port 22 open to the whole internet is wider than Week 05, and it is
deliberate: GitHub's build machines have no fixed address, so there is no
narrower rule that lets them in. What that costs is spelled out in the
README.

✓ The instance reaches **Running**, with a **Public IPv4 address**.

### 6 · Write the address down

Copy the **Public IPv4 address** from the Instances page. That number is
`YOUR-IP` for the rest of this sheet, and in a moment it goes into a GitHub
secret.

> **Do not stop the instance.** A stopped instance gives its address back,
> and starting it again gets you a different one — at which point the
> pipeline keeps deploying to an address that is no longer yours. If that
> does happen, it is not fatal: update the `EC2_HOST` secret with the new
> number and push again.

### 7 · Send your keys over, and walk in

In the folder where the `.pem` landed — usually `~/Downloads`:

```bash
chmod 400 YOUR-KEY.pem
scp -i YOUR-KEY.pem ~/buildrbank-cicd/.env ubuntu@YOUR-IP:~
ssh -i YOUR-KEY.pem ubuntu@YOUR-IP
```

`scp` is `cp` over an SSH connection. Everything after the `:` is a path on
the server, so `:~` means the `ubuntu` user's home directory.

Adjust the path to `.env` if your project folder is somewhere else.

The first `ssh` asks *Are you sure you want to continue connecting?* — type
`yes`.

✓ The prompt changes to `ubuntu@ip-172-…`. You are on the server.

- `Permission denied (publickey)` — the user is `ubuntu`, not `ec2-user`.
- `connection timed out` — the port 22 rule from step 5.

### 8 · Install Docker

```bash
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker ubuntu
chmod 600 ~/.env
exit
```

Then log in again, because `usermod` only applies to a new session:

```bash
ssh -i YOUR-KEY.pem ubuntu@YOUR-IP
docker ps
```

✓ `docker ps` runs without `sudo` and prints an empty table.

> `curl: (3) URL rejected` means your paste lost a slash. It is `https://` —
> two slashes.

### 9 · Start it once, by hand

Your Week 05 image is still on Docker Hub. Nothing needs building to get
back on the air — and this is the last time you type any of it.

```bash
docker login
docker pull YOUR-NAME/buildrbank:v1
docker run -d --name bank --restart unless-stopped -p 8000:8000 --env-file ~/.env YOUR-NAME/buildrbank:v1
curl -sS http://localhost:8000/health
```

✓ `/health` answers `"status":"ok"`, and `http://YOUR-IP:8000` opens the
chat page in your browser.

> **`not found` on the pull?** Your Week 05 image may carry a different tag.
> Open `hub.docker.com/r/YOUR-NAME/buildrbank/tags` and use whatever is
> listed — any image of this app will do, because step 14 replaces it.

> **Container exits straight away?** `docker logs bank` — it is almost
> always a missing key in `~/.env`.

---

## Part C · Tell GitHub who you are

### 10 · A Docker Hub token

Docker Hub → **Account settings** → **Personal access tokens** →
**Generate new token**. Permissions: **Read & Write**.

Copy it now. You cannot read it again.

> A read-only token still *logs in* successfully and only fails at the push,
> so the error arrives late and blames the wrong step.

### 11 · Three secrets

Your fork → **Settings** → **Secrets and variables** → **Actions** →
**New repository secret**. Three times:

| Name | Value |
|---|---|
| `DOCKERHUB_TOKEN` | the token from step 10 |
| `EC2_HOST` | your server's address. Just the number — no `http://`, no `:8000` |
| `EC2_SSH_KEY` | the entire key file, `-----BEGIN` and `-----END` lines included |

For the key, print it and select it:

```bash
cat YOUR-KEY.pem
```

> This puts your private key on the screen. If you are sharing your screen
> or standing in front of a projector, do it afterwards.
>
> Easier and safer if you have an editor open: open the `.pem` as a file,
> `Cmd/Ctrl+A`, `Cmd/Ctrl+C`. You cannot miss a line that way.

Three things go wrong here, and all three produce errors that name something
else:

1. **The `%` at the end.** Your terminal prints a `%` (or a `⏎`) right after
   `-----END RSA PRIVATE KEY-----`. That is the shell telling you the file
   has no final newline. **It is not part of the key — do not copy it.**
2. **The missing newline it is telling you about.** After pasting into the
   secret box, **press Enter once** so the cursor sits on an empty line
   below `-----END`. The deploy job cannot read a key that stops dead on the
   last dash.
3. **The `-----END` line itself.** Select all the way to the end. Stopping
   one line short is the single most common mistake on this page.

Get any of them wrong and the deploy job says `ssh: no key found` or
`invalid format` — neither of which mentions a newline.

### 12 · One variable

Same page, the **Variables** tab next door → **New repository variable**:

| Name | Value |
|---|---|
| `DOCKERHUB_USER` | your Docker Hub username |

That tab is the whole lesson. A username is not a secret — it is printed on
every image you have ever pushed. A token is. GitHub built the distinction
into the furniture.

> Until `DOCKERHUB_USER` is set, build and deploy say **skipped** rather
> than failing. A freshly forked repo shows a green tick and greyed-out
> boxes, not a red cross.

---

## Part D · The pipeline

### 13 · Fill in the five TODOs

Open `.github/workflows/pipeline.yml`. Five lines say `TODO`, each with a
few sentences of reading above it.

| | What goes there | Answer |
|---|---|---|
| 1 | the secret holding your Docker Hub token | `DOCKERHUB_TOKEN` |
| 2 | a second image tag, the commit's own id | `${{ env.DOCKERHUB_USER }}/${{ env.IMAGE }}:${{ github.sha }}` |
| 3 | the secret holding your server's address | `EC2_HOST` |
| 4 | the secret holding your key | `EC2_SSH_KEY` |
| 5 | production's port mapping | `-p 8000:8000` |

TODO 2 goes **inside** the `tags:` list, on its own line under the first
tag. Keep the explanation above it outside the list — under `tags: |` every
line is a tag, including one that starts with `#`.

TODO 5 is a question, not a copy. `deploy-dev` just above it says
`-p 8001:8000`. The number on the left is the door on the server; the number
on the right is the port the app listens on inside the container, and that
never changes. dev answers on 8001 — so which door does production answer
on?

**YAML has no brackets: indentation is the syntax.** Keep exactly the spaces
that were already on the line you are replacing.

### 14 · Push, and watch it run

```bash
git add .github/workflows/pipeline.yml
git commit -m "Fill in the pipeline"
git push
```

Open the **Actions** tab. Three jobs, in order: **Tests**, **Build and
push**, **Deploy to production** — and **Deploy to dev** greyed out, because
you pushed to `main`.

✓ About four minutes later, `http://YOUR-IP:8000` is answering again — this
time from a container nobody started by hand.

The first run is often red. Open it, read the log, find the line. That is
the skill this week is actually teaching.

---

## Part E · Make it yours

### 15 · Change the background colour, on dev

```bash
git switch -c dev
```

Open `app/index.html`. Near the top is the page's palette. Find every line
that says `--ground:` — **there are three**, because the page has a light
theme and a dark one:

```
line 24   --ground:#F4F5F3;     the light theme
line 47   --ground:#141F1C;     dark, when your laptop is in dark mode
line 54   --ground:#141F1C;     dark, chosen explicitly
```

`--ground` is the background the whole chat page sits on. Put a colour of
your own in all three — any hex you like. Change only one and half the room
will see no difference at all, because their laptop is in the other theme.

```bash
git add app/index.html
git commit -m "A background of my own"
git push -u origin dev
```

✓ Actions: **Tests**, **Build and push**, **Deploy to dev**. *Deploy to
production is grey.*
✓ `http://YOUR-IP:8001` — **your colour.**
✓ `http://YOUR-IP:8000` — **still the old one.**

That pair is the point of the whole week. One server, one push, two
containers — and `main` was never touched.

### 16 · Merge it, and watch it reach the real one

Your fork → GitHub offers **Compare & pull request** → click it.

1. **Create pull request**
2. Wait for the green tick on the checks at the bottom
3. **Merge pull request** → **Confirm merge**

Then the **Actions** tab: a new run on `main`, with **Deploy to
production** this time and **Deploy to dev** greyed out.

✓ `http://YOUR-IP:8000` — your colour, about four minutes later.

### 17 · Now change something else, and just push

Back on `dev`. Change the colour again, or the accent (`--jade`), or the
text beside the title. Commit, push, open the pull request, merge.

You will not open AWS. You will not type `docker`. You will not log into the
server.

**`git push` is the deploy.**

---

## Part F · Afterwards

### 18 · When you need yesterday back

Every build pushes two tags: the branch's name, which moves, and the
commit's id, which never does. So going back is not a rebuild — it is a
different tag.

Find the id: your fork → **Commits** → the copy icon beside the commit you
want. Use the full 40-character id, not the short one GitHub displays.

```bash
ssh -i YOUR-KEY.pem ubuntu@YOUR-IP
docker rm -f bank
docker run -d --name bank --restart unless-stopped -p 8000:8000 --env-file ~/.env YOUR-NAME/buildrbank:THE-COMMIT-ID
```

✓ The old page is back in about thirty seconds, with no build and no push.

> The next push to `main` deploys `:main` again and overwrites this. Rolling
> back buys you time to fix the problem properly; it does not fix it.

### 19 · Shut it down when the course ends

EC2 → Instances → select → **Instance state** → **Terminate instance**.

A terminated instance stops costing anything, and its address goes back to
AWS. Nothing else to release.

---

## If it breaks

| What you see | Where to look |
|---|---|
| `cp: .env: Not a directory` | a path with a space in it, unquoted. See step 3. |
| `curl: (3) URL rejected` | your paste lost a slash. It is `https://`. |
| `unknown shorthand flag: 'n' in -name` | your paste lost a dash. It is `--name`, two dashes. |
| `curl` prints nothing at all | `-s` hides errors. Use `curl -sS`. |
| `ssh: no key found` in the deploy job | `EC2_SSH_KEY` has no final newline. Re-paste and press Enter after `-----END`. See step 11. |
| `invalid format` in the deploy job | `EC2_SSH_KEY` is missing the `-----BEGIN` or `-----END` line — or you copied the `%`. |
| `Permission denied (publickey)` | right key, wrong user. It is `ubuntu`, not `ec2-user`. |
| `connection timed out` on ssh | port 22 is still restricted to *My IP*. See step 5. |
| `insufficient scopes` on the push | the Docker Hub token is read-only. Generate a new one with **Read & Write**. |
| Build and deploy say **skipped** | `DOCKERHUB_USER` is not set. Variables tab, not Secrets. |
| Deploy is green, the page is unchanged | hard-refresh the browser, and check you opened the right port — 8001 for dev, 8000 for main. |
| Deploy is green, the site is down | the container started and died. `docker logs bank`. Almost always a missing key in `~/.env`. |
| `docker: invalid reference format` | an option lost a dash, so the image name was read as something else. See step 13. |
| It worked yesterday, now nothing deploys | the instance was stopped and restarted, so its address moved. Update `EC2_HOST`. See step 6. |
| It answers, but knows nothing | `seed-2-knowledge.sql` did not finish. Re-run it in the SQL editor. |

Read the log **before** you change anything. The first error is the real
one.
