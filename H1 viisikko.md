# H1 Viisikko

## Käyttöympäristö

Prosessori: AMD Ryzen 5 5500H

RAM: 8 GB DDR4

Näytönohjain: NVIVIA GeForce RTX 2050

OS: Windows 10

VM: Linux Debian 12 bookworm

## x) Lue ja tiivistä. 

### Karvinen 2023: Run Salt Command Locally 

- Saltin pää tarkoitus on yleensä ohjata suuria määriä orjakoneita verkon yli.
- Tärkeimpiä tilakomentoja ovat `pkg` , `file` , `service` , `user` & `cmd`

### Karvinen 2018: Salt Quickstart – Salt Stack Master and Slave on Ubuntu Linux 

- Ei ole väliä ovatko orjakoneet esim. NATin takana, palomuurin takana, tai tuntemattomassa osoitteessa. Niitä voi hallita silti. Ainoastaan pääpalvelimen pitää olla julkinen.
- Verkossa on yksi pääpalvelin eli "master" ja useita orjakoneita eli "slaves".

### Karvinen 2006: Raportin kirjoittaminen 

- Raportointi tarkoittaa, että kirjoitat täsmällisesti mitä teit ja mitä siitä seuraa.
- Raportti tulee olla **Toistettava** , **Täsmällinen** , **Helppolukuinen** ja **Täytyy viitata lähteisiin**

### WMWare Inc: Salt Install Guide: Linux (DEB)

#### Saltin asennus

  1. Luo tarvittava hakemisto ja lataa julkinen avain.

         # Keyrings hakemisto:
         $ mkdir -p /etc/apt/keyrings
         # Julkisen avaimen lataus
         $ curl -fsSL https://packages.broadcom.com/artifactory/api/security/keypair/SaltProjectKey/public | sudo tee /etc/apt/keyrings/salt-archive-keyring.pgp

  2. Päivitä pakettiluettelo

         $ sudo apt-get update

  3. Lataa salt-minion, salt-master tai joku muita salt-komponentteja.

## Lähteet

 
