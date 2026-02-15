# OpenCloud

File cloud powered by [OpenCloud](https://opencloud.eu/).

## DNS

Add a DNS entry for:

* `opencloud.ferguson.fr` → your server IP

If using pihole for local DNS, add this in **Local DNS → DNS Records**.

## Setup

1. Edit `.env` and set a strong `IDM_ADMIN_PASSWORD`.
2. Start the service:

```sh
docker compose up -d
```

3. Access at `https://opencloud.ferguson.fr` and log in with user `admin` and the password you set.
