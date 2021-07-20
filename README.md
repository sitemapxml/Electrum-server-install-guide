# Electrum Server
###### Postupak instalacije

Ovaj dokument opisuje kompletan proces instalacije i konfigurisanja svih neophodnih komponenti za rad `Electrum` servera.

Pre početka instalacije `Electrum` servera potrebno je imati konfigurisane sledeće komponente:

- [rust](https://www.rust-lang.org/tools/install) sa [nginx](https://docs.nginx.com/nginx/admin-guide/web-server/reverse-proxy/)-om
- [Bitcoin Core](https://bitcoin.org/en/full-node) (v0.16+)
- [Electrum wallet](https://electrum.org/#download) (v3.3+)

## Instalacija Rust kompajlera

```
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```
Nakon pokretanja skripte za instalaciju potrebno je odabrati opciju 1
`Proceed with installation`

Nakon završene instalacije potrebno je podesiti `environment` varijablu unutar `.bashrc` fajla:

```
nano ~/.bashrc
```
```
# Dodati na kraju
source $HOME/.cargo/env
```

Izaći iz terminala i restartovati SSH sesiju (ukoliko se koristi ssh)

Da se uverimo da sve radi kako treba:
```
rustup --version
```

Što kao rezultat treba da vrati:

```
rustup 1.24.3 (ce5817a94 2021-05-31)
info: This is the version for the rustup toolchain manager, not the rustc compiler.
info: The currently active `rustc` version is `rustc 1.53.0 (53cb7b09b 2021-06-17)`
```

I za `cargo`:
```
cargo --version
```

Što kao rezultat treba da vrati:

```
cargo 1.53.0 (4369396ce 2021-04-27)
```

## Podešavanje NGINX http servera
Instalacija:

```
apt install nginx
```

Obrisati sadržaj postojećeg konfiguracionong fajla:

```
cat /dev/null > /etc/nginx/default
```

Prekopirati konfiguraciju ispod i prilagoditi postojećem okruženju:

```
# http server block
server {
    listen 80;
    listen [::]:80;

    server_name example.com;

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_buffering off;
        proxy_set_header X-Real-IP $remote_addr;
    }
}

```
Zatim podesiti `SSL`:

```
apt install python3-certbot-nginx
certbot --nginx
```
> NAPOMENA: Za konfigurisanje ssl sertifikata za www pod-domen potrebno je podesiti novi server blok u kome server_name sadrži www.example.com ili odvojeni fajl sa sadržajem kao u primeru iznad. U slučaju odvojenog konfiguracionog fajla potrebo je napraviti meku vezu do direktorijuma /etc/nginx/sites-enabled 
>`ln -s /etc/nginx/sites-available/www.default /etc/nginx/sites-enabled/www.default`
>`systemctl reload nginx`

Nakon uspešno kreiranog SSL sertifikata, `certbot` će dodati server blok kao u primeru ispod:

```
# https
server {
    server_name example.com;

    listen [::]:443 ssl ipv6only=on;
    listen 443 ssl;
    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_buffering off;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

Postupak prilagođavanja (potrebno je izmeniti):
 + `server_name`
 + `ssl_certificate`
 + `ssl_certificate_key`

Za dobijanje informacija o putanjama instaliranih restifikata pokrenuti komandu:

```
certbot certificates
```

Za potrebe testiranja može se dodati u prvi server blok:
```
# Ukloniti komentare za root, index i location direktive
# samo za potrebe testiranja
#   root /var/www/html;
#   index index.html;
#   location / {
#               try_files $uri $uri/ =404;
#   }
```

## Instalacija Test aplikacije
Poželjno je i korisno testirati konfiguraciju pomoću demo aplikacije pisane u `Rust`-u Za potrebe testiranja koristićeno aplikaciju [seed-rs-realworld](https://github.com/seed-rs/seed-rs-realworld.git)


Postupak je sledeći:

```
cd /home
git clone https://github.com/seed-rs/seed-rs-realworld.git www

cd www
rustup update
rustup target add wasm32-unknown-unknown

apt install pkg-config -y
cargo install --force cargo-make
cargo make all
cargo make serve

```

Nakon uspešne instalacije, aplikacija bi trebala da bude dostupna na domenu koji je podešen u `nginx` konfiguraciji.
Ukoliko iz bilo kog razloga aplikaciju nije moguće otvoriti preko porta `80` ili `443` otvoriti u pretraživaču `http(s)://example.com:8000`

## Instalacija Bitcoin Full node (Bitcoin-core)

Sa zvaničnog sajta [bitcoin-a](https://bitcoin.org) potrebno je [preuzeti](https://bitcoin.org/download/) poslednju verziju za `linux`

```
wget https://bitcoin.org/bin/bitcoin-core-0.21.1/bitcoin-0.21.1-x86_64-linux-gnu.tar.gz

# ukoliko na sistemu nije prisutan wget
curl -O https://bitcoin.org/bin/bitcoin-core-0.21.1/bitcoin-0.21.1-x86_64-linux-gnu.tar.gz

tar -xzvf bitcoin-0.21.1-x86_64-linux-gnu.tar.gz

# bez menjanja radnog direktorijuma pokrenuti komandu install:
install -m 0755 -o root -g root -t /usr/local/bin bitcoin-0.21.1/bin/*
```

Ovo će instalirati sledeće komande:

```
bitcoin-cli  bitcoin-qt  bitcoin-tx  bitcoin-wallet  bitcoind  test_bitcoin
```

U kombinaciji sa `bitcoin-cli` komandom moguće je koristiti sledeće komande:

```
getblockchaininfo
getnetworkinfo
getnettotals
getwalletinfo
stop
and
help
```

Sada je instalacija `bitcoind` servisa završena i potrebno je pokrenuti proces sinhronizacije sa bitcoin blockchain-om.

Pre pokretanja `bitcoin daemon`-a potrebno je uveriti se da na hard disku ima dovoljno prostora. Ukoliko je u upotrebi cloud `VPS` kod nekog od `IaaS` provajdera moguće je dodati prostor `VPS` serveru.

Trenutno je, za ovaj postupak, potrebno najmanje `350GB` prostora na disku.

Procedura zavisi od konkrente infrastrukture, ali uglavnom izgleda ovako:

```
# To get started with a new volume, you'll want to create a filesystem on it:
mkfs.ext4 "/dev/disk/by-id/scsi-Volume_Name_my-volume-name"

# Once the volume has a filesystem, you can create a mountpoint for it:
mkdir "/mnt/my-volume-name"

# Then you can mount the new volume:
mount "/dev/disk/by-id/scsi-Volume_Name_my-volume-name" "/mnt/my-volume-name"

# If you want the volume to automatically mount every time your Linode boots, you'll want to add a line like the following to your /etc/fstab file:

nano /etc/fstab
/dev/disk/by-id/scsi-Volume_Name_my-volume-name /mnt/my-volume-name ext4 defaults,noatime,nofail 0 2
```

## Pokretanje sinhronizacije
Za pokretanje bitcoin `daemon`-a:

```
bitcoind -datadir=/mnt/moja-particija -daemon

# ukoliko na osnovnoj particiji ima dovoljno prostora (nije preporučeno@reboot bitcoind -daemon):
bitcoind -daemon
```

Ovo će pokrenuti `bitcoind` servis u pozadini. Za standardno pokretanje ukloniti opciju `-daemon`

Za samostalno pokretanje nakon ponovnog pokretanja sistema:

```
crontab -e
# odablrati opciju 1

# na kraju fajla dodati:
@reboot bitcoind -daemon
```

Za pokretanje `bitcoin gui` aplikacije pokrenuti komandu:

```
bitcoin-qt
```
`Bitcoin gui` se može pokrenuti i preko `SSH` ako je instaliran [Windows X Server](https://sourceforge.net/projects/vcxsrv/)
