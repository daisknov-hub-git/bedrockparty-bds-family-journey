# All I Wanted Was for Three Family Members to Join My BDS

*— More than 20 days, an entire three-day weekend, and one ordinary DTP operator's struggle with BedrockParty and ChatGPT*

## Introduction

I'm not a programmer. I'm not a network engineer, either. I'm just a DTP (desktop publishing) operator who normally works with Illustrator and specialized production software.

What I wanted to do was very simple.

**Play Minecraft with three members of my family.**

I wanted to run a Bedrock Dedicated Server (BDS) on a Windows PC and have everyone join the same world from a Nintendo Switch or PS5, each using their own Microsoft account. That was all.

At least, that's what I thought when I started.

Before I knew it, I had installed Docker Desktop, set up Ubuntu through WSL, investigated BedrockParty and NetherNet Bridge, chased ICE candidates, examined Microsoft OAuth tokens and XUIDs, and finally started modifying Go source code.

There was a three-day holiday along the way. Three whole days with plenty of free time.

**I spent all three days doing this.**

The investigation lasted more than 20 days. For almost all of that time, ChatGPT was right beside me.

I'd paste a log. ChatGPT would suggest a possible cause. "Let's check this next." I'd try it. It would fail. I'd paste another log. Then we'd try something else.

Change one thing. Restart. Check. If it didn't work, undo the change.

We peeled away the problems one layer at a time.

This article is not a universal guide promising that copying these settings will make everything work. Minecraft, BDS, and BedrockParty all receive updates, and different environments may behave differently.

This is a record of what happened in *my* environment, how we isolated the problems, and how all three of us eventually connected to the same BDS at once. I'm including the failures as well as the successes.

If even one detail helps someone else who's stuck, that's enough.

## Chapter 1 — My Switch Won't Connect to BDS

First, I set up BDS on my Windows PC. The server started. I could create a world. But when I tried to connect from the Nintendo Switch, I hit the first wall.

Unlike the PC version, Minecraft on Switch doesn't provide a straightforward option to enter any IP address and connect to an arbitrary Bedrock server.

In other words, I couldn't simply do this:

```text
Nintendo Switch
      │
      │ Enter the server IP directly
      ✕
      │
      ▼
     BDS
```

That's how I discovered BedrockParty, followed by NetherNet Bridge.

At the time, my understanding was roughly, "So this might let the Switch connect to BDS?"

Docker, WSL, OAuth, and Go hadn't entered the picture yet.

**Life was peaceful.**

## Chapter 2 — Docker, WSL, and Ubuntu

To run BedrockParty and the Bridge, I installed Docker Desktop on Windows. Then came WSL, Ubuntu, Docker Compose, and containers.

Suddenly, I was dealing with a vocabulary I almost never encounter in my DTP work.

Here is a greatly simplified diagram of the final setup:

```text
Nintendo Switch / PS5
          │
          │ Via BedrockParty
          ▼
┌───────────────────────────────────┐
│ Windows PC                        │
│                                   │
│ Docker Desktop / WSL2 / Ubuntu    │
│       │                           │
│       ▼                           │
│ NetherNet Bridge ────────┐       │
│                          ▼       │
│                         BDS      │
└───────────────────────────────────┘
```

*This is a conceptual diagram of the communication path, not an exact depiction of the relationships among Docker Desktop, WSL2, and Ubuntu.*

Of course, I didn't understand the whole picture back then. I worked through it with ChatGPT, one step at a time.

Start the Bridge. Check the logs. Start BDS. Check its logs. Try connecting from Switch.

No connection.

Check the logs again.

Around this point, I naturally developed a rule for troubleshooting:

**Never change several settings at once.**

Change one thing, restart, check the result, and undo it if necessary. If you change several things together, even a successful result won't tell you which change mattered.

Looking back, perhaps that was the one part of the process where I already thought like a DTP operator.

## Chapter 3 — BedrockParty Appeared. But I Still Couldn't Join.

After enough trial and error, the BedrockParty world finally appeared on the Switch.

That was a big step forward. I'd gone from seeing nothing to seeing the world listed.

I immediately tried to connect.

It didn't work.

BDS was running. The Bridge was running. The Switch could see the world. Still, I couldn't get in.

I started examining the NetherNet-related logs in more detail. I should be clear: I'm not someone who can read a log and immediately understand it. My starting point was to paste it into ChatGPT and ask, "What does this mean?"

While inspecting the SDP Answer returned by BDS, we found this:

```text
ICE candidates: 0
```

BDS was there. The Bridge was there. The Switch was waiting.

**The only thing missing was every single candidate.**

At the time, my BDS `server.properties` specified a UDP port range:

```properties
server-udp-ports=49152-49200
```

We tried commenting out that setting:

```properties
#server-udp-ports=49152-49200
```

After restarting BDS and checking again, this:

```text
ICE candidates: 0
```

became this:

```text
ICE candidates: 5
```

And the connection through the Bridge finally progressed.

I want to be precise here. I'm **not** claiming that setting `server-udp-ports` always causes problems.

In my environment, enabling that setting resulted in zero ICE candidates; commenting it out produced five, and the connection started working. That's what I actually observed.

## Chapter 4 — The Switch Connected. Then Came the Next Problem.

I changed the setting and restarted BDS. I started the Bridge and selected the BedrockParty world on the Switch.

And finally:

**The Switch connected to BDS.**

A connection that had previously existed only in logs appeared on an actual Minecraft screen.

I'd cleared a major hurdle. Next was the PS5. It connected, too.

But using the same Microsoft/Xbox account on both the Switch and PS5 wasn't suitable for our goal of playing together reliably as a family.

Fine. We'd use a separate Microsoft account for each family member.

The parent would be the parent. The child would be the child.

Perfectly ordinary, right?

What came next was anything but ordinary.

## Chapter 5 — A Child Went In, but a Father Came Out

I connected from the Switch using a child's Microsoft account and inspected the incoming Minecraft Identity at the Bridge.

The real gamertags and XUIDs have been replaced with placeholders in this public version.

```text
IDENTITY_DEBUG
DisplayName=<CHILD_A_GAMERTAG>
XUID=<CHILD_A_XUID>
```

Correct. At the entrance, the Bridge recognized the player as Child-A.

But when I examined the Multiplayer Token used on the way to BDS, I found this:

```text
MULTIPLAYER_TOKEN_DEBUG
xname=<PARENT_GAMERTAG>
xid=<PARENT_XUID>
```

Put the two side by side:

```text
Incoming Minecraft Identity
  DisplayName = <CHILD_A_GAMERTAG>
  XUID        = <CHILD_A_XUID>
```

```text
Backend Multiplayer Token
  xname = <PARENT_GAMERTAG>
  xid   = <PARENT_XUID>
```

A child at the entrance. The father at the exit.

**A child went in, but somehow a father came out.**

If that makes no sense to you, I understand.

It didn't make sense to me at first, either.

But these logs became the turning point.

## Chapter 6 — Following the Identity

The incoming Minecraft Identity at the Bridge was correct. So perhaps the solution was to preserve that Identity all the way through to BDS.

We examined how Identity was handled, including in gophertunnel, and tried a patch to preserve the incoming Identity.

Modify. Build. Launch. Connect. Check logs.

It wasn't enough.

The incoming Identity was Child-A, but the backend Multiplayer Token still belonged to the parent.

That was when we began to understand that **preserving the Minecraft Identity and authenticating to BDS with the correct Microsoft Identity were separate issues**.

I didn't understand this architecture from the beginning. I gave ChatGPT the logs, traced the code paths, added debug output, and connected again. Over and over.

Eventually, we found a critical part of the Bridge's Microsoft authentication logic.

Conceptually, the original process looked like this:

```go
token, err := auth.RequestLiveToken()
tokenSource := auth.RefreshTokenSource(token)
```

The Bridge authenticated a Microsoft account at startup and used that TokenSource for backend connections.

For one person, that was fine.

But we had three family members with three separate Microsoft accounts.

```text
Parent  ───────────┐
Child-A ───────────┼──→ NetherNet Bridge
Child-B ───────────┘           │
                              ▼
                       Shared TokenSource
                              │
                              ▼
                             BDS
```

With that arrangement, whoever arrived at the entrance would leave using the account authenticated by the Bridge.

At last, the strange behavior made sense.

The child hadn't magically transformed into the father halfway through. **The Bridge was using the father's Microsoft token at the exit.**

To confirm, we authenticated the Bridge using the child's Microsoft account instead:

```text
MULTIPLAYER_TOKEN_DEBUG
xid=<CHILD_A_XUID>
xname=<CHILD_A_GAMERTAG>
```

Authenticate the Bridge as the parent, and the backend becomes the parent. Authenticate as the child, and the backend becomes the child.

What we needed was to select the backend Microsoft token based on the player entering the Bridge.

## Chapter 7 — A DTP Operator Starts Editing Go Source Code

Let's establish where I was at this point.

I'm a DTP operator. My everyday tools are Illustrator and a specialized typesetting system.

I can't write Go. I'm not an OAuth expert.

Yet there I was, on my day off, researching how to switch TokenSources according to the incoming XUID.

**How did my life come to this?**

Of course, I didn't write an entire program from scratch. With ChatGPT, I worked through each step: "This log suggests we should inspect here," "Add debug output here," "Change this process."

The idea itself was simple.

Each player has a different XUID. So use the XUID as a key to select that player's own Microsoft OAuth token.

```text
Parent  ── XUID A ── Token A ──┐
Child-A ── XUID B ── Token B ──┼──→ Bridge → BDS
Child-B ── XUID C ── Token C ──┘
```

We added logic to choose a token file from the XUID. With values replaced for publication, the concept looked like this:

```go
func tokenPathForXUID(xuid string) (string, bool) {
    switch xuid {
    case "<PARENT_XUID>":
        return "/data/nethernet/<PARENT_XUID>.json", true
    case "<CHILD_A_XUID>":
        return "/data/nethernet/<CHILD_A_XUID>.json", true
    case "<CHILD_B_XUID>":
        return "/data/nethernet/<CHILD_B_XUID>.json", true
    default:
        return "", false
    }
}
```

*This is an illustrative example, not a complete drop-in patch. No real XUIDs or authentication files are included.*

In other words:

```text
Incoming XUID
      ↓
tokenPathForXUID()
      ↓
OAuth token for that XUID
      ↓
Multiplayer Token
      ↓
BDS
```

Did I understand every line of code? No.

But I checked what we'd changed, and what happened in the logs as a result.

One change at a time.

## Chapter 8 — This Time, the Child Came Out as the Child

We modified and rebuilt the Bridge, started it, and connected from the Switch using the child's account.

At the entrance:

```text
IDENTITY_DEBUG
DisplayName=<CHILD_A_GAMERTAG>
XUID=<CHILD_A_XUID>
```

At the backend:

```text
MULTIPLAYER_TOKEN_DEBUG
xname=<CHILD_A_GAMERTAG>
xid=<CHILD_A_XUID>
```

They matched.

```text
Incoming Identity : Child-A
Backend Identity  : Child-A
```

**This time, the child made it all the way through as the child.**

And connected to BDS.

Written down, it sounds like the most obvious thing in the world. It took us a long time to get to that obvious result.

But there were three of us.

When the third Microsoft account tried to connect, we got a message along these lines:

```text
no Microsoft token configured for player
DisplayName=<CHILD_B_GAMERTAG>
XUID=<CHILD_B_XUID>
```

Early on, I would have seen that and thought, "It still doesn't work."

By now, I could read just enough of the logs to understand: **there's no Microsoft token registered for this XUID yet**.

So we added a way to register a token for each account by specifying its XUID:

```bash
docker compose -f compose.win.yml run --rm nethernet-bridge \
  --register-token <XUID>
```

We registered the third token and restarted the Bridge.

Then all three of us connected.

In the BDS console, I entered:

```text
list
```

With the names replaced for publication, the response was:

```text
There are 3/10 players online:
<CHILD_B_GAMERTAG>, <PARENT_GAMERTAG>, <CHILD_A_GAMERTAG>
```

**3/10 players online.**

Three people. Three separate Microsoft accounts. One BDS. One world.

The thing I'd wanted to do from the beginning had finally happened.

## Chapter 9 — Working Today Isn't Enough

Three simultaneous connections worked. But this was supposed to be a family server we'd use regularly. I didn't want to repeat Microsoft authentication every time we started it.

So we wrapped the TokenSource and added a process to save refreshed tokens to a separate file for each player.

```go
type persistingTokenSource struct {
    src  oauth2.TokenSource
    path string
    mu   sync.Mutex
}
```

```go
func (p *persistingTokenSource) Token() (*oauth2.Token, error) {
    p.mu.Lock()
    defer p.mu.Unlock()

    token, err := p.src.Token()
    if err != nil {
        return nil, err
    }

    if err := saveLiveToken(p.path, token); err != nil {
        return nil, err
    }

    return token, nil
}
```

On the Bridge side:

```go
tokenSource := &persistingTokenSource{
    src:  auth.RefreshTokenSource(token),
    path: tokenPath,
}
```

We mounted the token storage directory into Docker:

```yaml
volumes:
  - "./secrets/nethernet:/data/nethernet:rw"
```

And added this to `.gitignore`:

```gitignore
secrets/
```

This is especially important when publishing anything. **Never publish real OAuth tokens, refresh tokens, device codes, private keys, or similar secrets.**

In this article, I've also removed or anonymized Microsoft/Xbox account names, XUIDs, home LAN IP addresses, Windows usernames, and other information that could identify my family or home setup.

## Chapter 10 — Then the Network Disappeared

When you spend more than 20 days troubleshooting, things unrelated to the Bridge can go wrong, too.

At one point, the network stopped working.

Not just Minecraft.

**The network itself was broken.**

I'd spent so much time changing Docker, WSL, ports, the Bridge, and networking settings that I genuinely panicked. Had I touched something I really shouldn't have?

And of course, the PC whose network I needed to investigate was the PC whose network wasn't working.

So I picked up my smartphone and retreated to the ChatGPT mobile app.

Standing in front of the PC, I reported what I could see from my phone, checked one thing, sent the result, and checked the next. Somehow, while trying to fix Minecraft connectivity, I'd ended up repairing the PC's own network.

**The problem had leveled up.**

Fortunately, the network recovered.

Then, as if nothing had happened, I went right back to troubleshooting Minecraft.

Yes, I was still doing this.

## Chapter 11 — The Final Boss: Starting Everything Is a Pain

Three people could connect. Tokens could be saved.

Technically, we were finished.

But every time we wanted to play, I had to launch Docker Desktop, start BDS, and run the Bridge through WSL/Ubuntu. When we finished, I had to send `stop` to BDS, stop the Bridge, and shut down Docker Desktop.

That was too much work for every session.

After coming this far, I wanted one-click operation.

So I created two desktop batch files:

```text
Minecraftサーバー起動.bat
Minecraftサーバー終了.bat
```

The startup sequence was:

```text
Double-click
      ↓
Docker Desktop
      ↓
BDS
      ↓
NetherNet Bridge
      ↓
Ready
```

But even here, something went wrong.

Initially, the script checked whether UDP port 19132 was in use to decide whether BDS was running. The batch file reported `BDS OK`—even though BDS wasn't running.

So I checked:

```powershell
Get-NetUDPEndpoint -LocalPort 19132 |
    Select-Object LocalAddress,LocalPort,OwningProcess
```

A process really was using port 19132.

When I investigated the PID, it belonged to:

```text
com.docker.backend
```

The port was in use, all right.

Just not by BDS.

**You're not the one I'm looking for.**

We changed the startup check to look for `bedrock_server.exe` itself instead of relying on the port.

The shutdown side needed care, too.

It would have been easy to use `taskkill`, but BDS manages a world. I wanted to send the same command I'd type into its console:

```text
stop
```

And wait for a clean shutdown, including messages such as `Server stop requested.`, `Stopping server...`, and `Quit correctly`.

So we launched BDS from PowerShell while retaining its StandardInput, and used a Windows named pipe to pass `stop` from the shutdown batch file.

```text
Minecraftサーバー終了.bat
             │
             │ "stop"
             ▼
      Windows Named Pipe
             │
             ▼
 MinecraftBDS-Control.ps1
             │
             ▼
      bedrock_server.exe
             │
             ▼
        Quit correctly
```

The test produced the equivalent of:

```text
Received stop from the shutdown batch file.
Server stop requested.
Stopping server...
Quit correctly
```

BDS shut down normally.

Afterward, we stopped the Bridge and shut down Docker Desktop with:

```bash
docker desktop stop
```

At last, it became a practical family server: double-click the startup batch file before playing, and double-click the shutdown batch file afterward.

## Chapter 12 — Even the Failures Were Useful

If I wrote down only the settings that worked, this article would be much shorter.

But in this case, the failures were valuable, too.

In my environment, `server-udp-ports=49152-49200` resulted in zero ICE candidates. Preserving only the incoming Identity didn't fix the multi-account problem; the backend Microsoft Multiplayer Token had to match as well.

Sharing one Microsoft TokenSource among all players meant they appeared as the same backend Identity. Using the same Microsoft account on Switch and PS5 wasn't suitable for our simultaneous family play, either.

Checking only UDP 19132 could mistake Docker's backend for BDS. And I didn't want to force-kill BDS carelessly.

Every time we solved one problem, another appeared behind it.

That was pretty much the entire experience.

In the end, the structure was:

```text
Parent  ── XUID A ── OAuth Token A ──┐
Child-A ── XUID B ── OAuth Token B ──┼──→ Bridge → BDS
Child-B ── XUID C ── OAuth Token C ──┘
```

If I had to express the core of the multi-account problem in one line, it was turning this:

```text
Incoming Identity ≠ Backend Microsoft Identity
```

Into this:

```text
Incoming Identity = Backend Microsoft Identity
```

That, I think, was the heart of the solution.

## Chapter 13 — When the Next Major Update Arrives

The setup works in my current environment.

But I don't expect it to keep working unchanged forever. Minecraft gets updates. BDS changes. BedrockParty and gophertunnel may change, too.

If a future major update changes the protocol, one day I may suddenly find myself saying:

"I can't connect again."

So I plan to preserve the currently working setup as my tested, approved version.

If something breaks, I'll check the Minecraft and BDS versions. Then the protocol, the Bridge, ICE candidates, incoming Identity, backend Multiplayer Token, and per-XUID tokens.

There's one major difference between now and 20 days ago.

At first:

**I didn't even know what I didn't know.**

When an error appeared, I didn't know where to look. Even when I read a log, I couldn't tell whether it mattered.

Now it's different.

The next error won't send me all the way back to square one.

**This time, I know where to look.**

Of course, I'd prefer the next major update to break absolutely nothing.

That would be best.

## Final Chapter — And So, the Legend Begins...

More than 20 days.

Docker, WSL, Ubuntu, BedrockParty, NetherNet, ICE, SDP, XUID, OAuth, Multiplayer Token, Go, named pipes.

Before I started, I knew almost none of those things.

I had just one goal.

**Play Minecraft with three members of my family.**

That was all.

There was a three-day holiday in the middle.

I used every minute of it.

Change a setting. Restart. Fail. Read the log. Paste it into ChatGPT. Figure out what to inspect next. Try one thing. Undo it if it doesn't work.

At one point, the network itself failed and I genuinely panicked. I couldn't investigate from the PC anymore, so I switched to the ChatGPT smartphone app and kept working on recovery.

I even edited Go code.

Of course, that doesn't mean I've become a Go programmer or an OAuth expert.

But there was one rule I followed all the way through:

**Make one change. Check one result.**

Then one day, in the BDS console, I typed:

```text
list
```

And got this:

```text
There are 3/10 players online:
<CHILD_B_GAMERTAG>, <PARENT_GAMERTAG>, <CHILD_A_GAMERTAG>
```

Three people.

Three separate Microsoft accounts.

The same BDS. The same world.

When a child entered the Bridge, the child finally came out as the child.

At last.

I'm not a programmer. I'm not a network engineer.

**I'm a DTP operator.**

So why was I able to keep debugging for more than 20 days?

Why could I fix one thing, check it, fix another, and check again, over and over?

When I thought about it after everything was finished, I began to understand.

It's what I do at work, too.

I submit the first proof. Corrections come back marked in red. I make the changes and submit the next proof. More corrections come back. I fix them again.

Has any text shifted? Is an image missing? Does something differ from the instructions?

Check each item. Fix what is wrong. Repeat until it is right.

This time was no different.

Instead of a printed page, it was a server.

Instead of Illustrator, it was Docker and Go.

And:

**Instead of red correction marks, I got error logs.**

Maybe that's why I could keep going, even when almost everything was unfamiliar.

If someone asked me:

"How did you manage to be so patient while debugging?"

My answer would be simple.

**Because I'm an operator.**

More than 20 days.

First proof.

Corrections.

Second proof.

More corrections.

Back again, fix it again, check it again.

And finally, three names appeared together on BDS.

Now it's ready to go out.

**Approved for publication.**
