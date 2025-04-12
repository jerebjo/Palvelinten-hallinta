# h3 Infraa koodina

## Käyttöympäristö

Prosessori: AMD Ryzen 5 5500H

RAM: 8 GB DDR4

Näytönohjain: NVIVIA GeForce RTX 2050

OS: Windows 10

VM: Linux Debian 12 bookworm 64-bit

## a) Hei infrakoodi! Kokeile paikallisesti (esim 'sudo salt-call --local') infraa koodina. Kirjota sls-tiedosto, joka tekee esimerkkitiedoston /tmp/ -kansioon.

Tehtävää varten minulla oli jo luotuna aiempien tehtävien kaksi vagrant konetta. Ensin koneet päälle:

    $ Vagrant up

![Vagrant up](Kuvat/twohost.png)

Sitten kirjauduin slave-koneelle käyttäen ssh kirjautumista, koska tiedostot tulee ajaa paikallisesti: 

    $ vagrant ssh t002

![ssh kirjautuminen minionille](Kuvat/minionlogin.png)

