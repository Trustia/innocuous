# NitroMTA 0.2.1 Execution & Delivery Report

- **Installation & Exact Version**: Downloaded the official Debian package `nitromta_0.2.1_amd64.deb` from the GitHub release (`https://github.com/nitrosend/nitromta/releases/download/v0.2.1/nitromta_0.2.1_amd64.deb`) and installed it via `dpkg -i nitromta_0.2.1_amd64.deb`. Verified the binary version with `nitromta --version`, which output `nitromta 0.2.1`.
- **OS & Architecture**: Ubuntu 22.04.5 LTS (`jammy`) on WSL2, kernel `Linux 6.18.54.1-microsoft-standard-WSL2 x86_64` (`amd64`).
- **Exact Commands & `config.toml` Changes**:
  1. Created a dedicated unprivileged user and native ext4 workspace (`/home/nitromta/work`) because `nitromta run` refuses to run as `root` (`uid 0`) and rejects key files located on DrvFs `/mnt/c` mounts where POSIX mode bits default to `0777`.
  2. Initialized the configuration:
     ```bash
     su -s /bin/bash nitromta -c "cd /home/nitromta/work && nitromta setup --config config.toml --capture"
     ```
  3. Edited `/home/nitromta/work/config.toml` for live delivery:
     - Removed the entire `[capture]` section (`dir = "./capture"`).
     - Under `[delivery]`, set `port = 2525` and `dane = false` (replacing the commented defaults `# port = 25` and `# dane = true`, since the local WSL2 forwarder at `10.255.255.254:53` does not validate DNSSEC locally).
     - Set `[log]` `level = "debug"` to capture full DNS and SMTP attempt telemetry in `nitromta.log`.
  4. Started the engine in the background as the `nitromta` user:
     ```bash
     su -s /bin/bash nitromta -c "cd /home/nitromta/work && nohup nitromta run --config config.toml > nitromta.log 2>&1 &"
     ```
  5. Generated a 2048-bit RSA DKIM private key and provisioned the account, identity, and SMTP/SES credentials over `./admin.sock`:
     ```bash
     openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:2048 -out /home/nitromta/work/dkim.pem
     chown nitromta:nitromta /home/nitromta/work/dkim.pem && chmod 600 /home/nitromta/work/dkim.pem
     su -s /bin/bash nitromta -c 'cd /home/nitromta/work && nitromta admin --config config.toml <<EOF
     {"op":"put_account","id":"acct1","name":"Trustia Test"}
     {"op":"put_identity","account":"acct1","domain":"trustia.com.tr","selector":"s1","private_key_file":"./dkim.pem"}
     {"op":"create_login","account":"acct1","name":"smtp-user"}
     EOF'
     ```
  6. Submitted a test email over local SMTP (`127.0.0.1:2587`) with `STARTTLS` and `AUTH PLAIN` using the returned login username and generated secret (redacted), `From: hello@trustia.com.tr`, `To: proof@nitromta.com`, and `Reply-To` set to our operator mailbox.
  7. Verified delivery via `nitromta explain --config config.toml --message 00c691d9a1e00000` (and `--delivery 00c691d9a2200001`):
     ```text
     delivery 00c691d9a2200001 of message 00c691d9a1e00000 to proof@nitromta.com
       state      finished, delivered, at 2026-11-05T11:13:03.377Z
       expires    2026-11-08T11:12:57.617Z
       attempts   1, 0 of them uncertain
     ```
- **Minutes from Install to Receipt**: Approximately **14 minutes** end-to-end from `dpkg -i nitromta_0.2.1_amd64.deb` to receiving the signed delivery receipt (`https://runx.ai/r/d53075a1a949bf294312915292d487cb83882cbcb7b2b59571e63b49428b6b64`) at `2026-11-05T11:13:05Z`.
- **Step That Slowed Us Most**: Running inside WSL2 where the default shell starts as `root` and `/mnt/c` uses Windows DrvFs permissions (`0777`). First, `nitromta setup` on `/mnt/c` failed strict key-file permission validation (`./keys` must be owner-only `0600`). Second, the `.deb` package creates the `nitromta` system user with `/usr/sbin/nologin` and home `/var/lib/nitromta` (which is not created on disk), so running commands as `nitromta` required creating `/home/nitromta/work` on native ext4 and invoking `su -s /bin/bash nitromta`.
- **Reproducible Defect / Documentation Gap**:
  1. **`nitromta explain` omits `last_reply` on delivered messages (`docs/reference/cli.md` discrepancy)**:
     - **Repro**: Send a message that succeeds with HTTP/SMTP `250` (`result: "delivered"`, `code: 250` in `nitromta.log`), then run `nitromta explain --config config.toml --message 00c691d9a1e00000`.
     - **Observed vs. Documented**: `docs/reference/cli.md` (`https://raw.githubusercontent.com/nitrosend/nitromta/main/docs/reference/cli.md`) states that `explain` prints the remote server's `last_reply` alongside `state`, `expires`, and `attempts`. However, on a successful delivery (`state: finished, delivered`), `explain` only outputs `state`, `expires`, and `attempts` and omits the `250` server reply string unless a failure/deferral occurred.
  2. **Debian package user setup (`docs/install.md` gap)**:
     - **Repro**: Install `nitromta_0.2.1_amd64.deb` on Ubuntu 22.04 (`dpkg -i nitromta_0.2.1_amd64.deb`) and run `su - nitromta`.
     - **Observed**: Prints `su: warning: cannot change directory to /var/lib/nitromta: No such file or directory` and `This account is currently not available` because the postinst script creates `nitromta` with `/usr/sbin/nologin` and without creating `/var/lib/nitromta`.
- **One Change We Would Make**: Have `nitromta explain` include the final `250 ...` SMTP response text (or remote queue ID) in `last_reply` even for `finished, delivered` deliveries so operators can correlate message delivery receipts with remote MX logs without grepping `nitromta.log` at `level = "debug"`.
