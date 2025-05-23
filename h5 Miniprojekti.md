# h5 Miniprojekti 

## Käyttöympäristö

Prosessori: AMD Ryzen 5 5500H

RAM: 8 GB DDR4

Näytönohjain: NVIVIA GeForce RTX 2050

OS: Windows 10

VM: Vagrant

## Miniprojekti

Aluksi mietin minkä tyyppisen projekin voisin tehdä. Halusin, että aihe liittyy peleihin tai elokuviin. Päätin ideoida tekoälyn kanssa ja lopulta päädyin valitsemaan aiheeksi elokuva triviapelin. Se tulee toimimaan saltin avulla, pelissä 2 tai useampi pelaaja(VM) kilpailevat toisiaan vastaan arvaamalla elokuvia eri kysymysten mukaan. 

## Koneidein asennus

Ensiksi laitoin koneet pystyyn luomalla vagrantfilen: 

    # Master-koneen scripti, jossa asennetaan salt valmiiksi
    $master_script = <<-MASTER_SCRIPT
    set -o verbose
    sudo apt-get update
    sudo apt-get install -y curl tree
    sudo mkdir -p /etc/apt/keyrings
    sudo curl -fsSL https://packages.broadcom.com/artifactory/api/security/keypair/SaltProjectKey/public | sudo tee /etc/apt/keyrings/salt-archive-keyring.pgp
    sudo curl -fsSL https://github.com/saltstack/salt-install-guide/releases/latest/download/salt.sources | sudo tee /etc/apt/sources.list.d/salt.sources
    sudo apt-get update
    sudo apt-get install -y salt-master
    sudo systemctl restart salt-master.service
    MASTER_SCRIPT
    
    # Minion-koneen scripti, jossa asennetaan salt valmiiksi ja otetaan yhteys Master-koneeseen.
    $minion_script = <<-MINION_SCRIPT
    set -o verbose
    sudo apt-get update
    sudo apt-get install -y curl tree
    sudo mkdir -p /etc/apt/keyrings
    sudo curl -fsSL https://packages.broadcom.com/artifactory/api/security/keypair/SaltProjectKey/public | sudo tee /etc/apt/keyrings/salt-archive-keyring.pgp
    sudo curl -fsSL https://github.com/saltstack/salt-install-guide/releases/latest/download/salt.sources | sudo tee /etc/apt/sources.list.d/salt.sources
    sudo apt-get update
    sudo apt-get install -y salt-minion
    echo -e 'master: 192.168.88.101' |sudo tee /etc/salt/minion
    sudo systemctl restart salt-minion.service
    MINION_SCRIPT
    
    Vagrant.configure("2") do |config|
      # Virtuaalikone käyttämään synkronoitua kansiota.
      config.vm.synced_folder ".", "/vagrant", disabled: true
      config.vm.synced_folder "shared/", "/home/vagrant/shared", create: true
      config.vm.box = "debian/bookworm64"
    
      # Master VM
      config.vm.define "master" do |master|
        master.vm.hostname = "master"
        master.vm.network "private_network", ip: "192.168.88.101"
        master.vm.provision "shell", inline: $master_script
      end
    
      # Slave (minion) VM
      config.vm.define "slave", primary: true do |slave|
        slave.vm.hostname = "slave"
        slave.vm.network "private_network", ip: "192.168.88.102"
        slave.vm.provision "shell", inline: $minion_script
      end
    end

Sitten kokeilin käynnistää: 

    $ vagrant up

![VM:t päällä](Pkuvat/vmsup.png)

Sitten vielä kirjauduin masterille ssh-yhteydellä: 

    $ vagrant ssh master

Seuraavaksi yritin pingata slave-konetta: 

![ping the slave](Pkuvat/pingslave.png)

Seuraavaksi piti hyväksyä slave-koneen avain: 

    $ sudo salt-key -L

![Avaimen tarkistus](Pkuvat/slavekey.png)

    $ sudo salt-key -A

![Avaimen hyäksyminen](Pkuvat/acceptslave.png)

Lopuksi vielä ping-testi, että kaikki toimii varmasti tähän asti: 

    $ sudo salt '*' test.ping

![ping test saltin yli](Pkuvat/pingtest.png)

## Lokaalitestaus minionilla

Aloitin luomalla uuden kansion: 

    $ sudo mkdir -p /srv/salt
    $ sudo nano /srv/salt/install_python.sls

Tein yksinkertaisen tiedoston jonka pitäisi asentaa python: 

![install python](Pkuvat/pythonladattu.png)

Lopuksi vielä ajoin lokaalisti, nähdäkseni tuleeko mitään ongelmia: 

    $ sudo salt-call --local state.apply install_python

![Pythonin lataus lokaalisti](Pkuvat/lokaaliajo.png)

## Mestarilta orjalle

Seuraavaksi testasin voiko saman toteuttaa master-koneella ja sitten käskeä orja tekemään sama toiminto. Tein saman nimisen tiedoston myös master-koneelle ja kokelin ajaa sen saltin yli: 

    $ sudo salt 'slave' state.apply install_python

![saltin yli](Pkuvat/mestariltaorjalle.png)

## Pelin rakennus


Seuraavaksi lähdin luomaan itse peliä. Ensiksi halusin, että pelin pystyy ladata kuka vain, joten vagrantfilen master-koneen synkronointiin piti tehdä pieni lisäys:

Eli lisäsin kohdan `master.vm.synced_folder "salt/", "/srv/salt", owner: "root", group: "root"`

      # Master VM
      config.vm.define "master" do |master|
        master.vm.hostname = "master"
        master.vm.network "private_network", ip: "192.168.88.101"
        master.vm.synced_folder "salt/", "/srv/salt", owner: "root", group: "root"
        master.vm.provision "shell", inline: $master_script
      end

### Rakenteen suunnittelu

Eli seuraavaksi suunnittelin tekoälyn kanssa rakenteen pelille. Tarkoituksena oli tehdä elokuvatrivia-peli, jossa käyttäjä saa monivalitakysymyksen ja oikeasta vastauksesta saa pisteen, jotka lasketaan yhteen pelin edetessä. Rakenne lyhyesti: 

- app.py: Flask sovelluksen pääohjelma, joka: 
  - Hallinnoi peli-istuntoa
  - Näyttää lomakkeen pelaajalle
  - Tarkistaa vastauksen
  - Näyttää palautteen ja lopputuloksen
- html-pohjat:
  - quiz.html: painikkeet vaihtoehdoille
  - feedback.html: näyttää menikö vastaus oikein
  - result.html: näyttää lopulliset pisteet ja mahdollisuuden aloittaa uudelleen
- init.sls: Saltin pääkonfiguraatiotiedosto, joka kertoo mitä asennetaan ja mihin:
   - Asentaa Apache2, WSGI-moduulin ja Flaskin
   - Kopioi trivia-pelin tiedostot oikeaan hakemistoon `/var/www/trivia`
   - Ottaa käyttöön oman `trivia.conf`-sivuston
   - Poistaa Apachen oletussivun `000-default.conf`
   - Käynnistää Apache-palvelun
- top.sls: Saltin "reittikartta", joka määrittää mitä tiloja (init.sls) ajetaan mille koneille.
- trivia.wsgi: WSGI-käynnistystiedosto, jota Apache käyttää Flask-sovelluksen ajamiseen.

#### app.py luominen

Siirryin master koneelle johon loin uuden kansion `trivia`

    $ mkdir trivia

Seuraavaksi tein uuden tiedoston `app.py`, jonka tarkoitus on hallinnoida sessiota:

    $ micro app.py

Pyysin chatGPT:ltä apua tiedoston sisällön luonnissa. Lopputulos näytti tältä: 

![app.py sisältö](Pkuvat/app.py.png)

#### Html-pohjat

Seuraavaksi loin html-pohjat ja tätä varten tein uuden kansion `trivia`-kansion sisälle nimeltä `templates`:

    $ mkdir templates

Ensin `quiz.html`, jossa näytetään itse pelikysely: 

    $ micro quiz.html

![quiz.html koodi](Pkuvat/quiz.png)

Seuraavaksi loin `result.html`, joka näyttää tuloksen ja mahdollistaa uudelleen pelaamisen:

    $ micro result.html
![result.html koodi](Pkuvat/result.png)

Lopuksi vielä `feedback.html`, joka kertoo pisteytyksestä: 

    $ micro feedback.html

![feedback.html koodi](Pkuvat/feedback.png)

#### Flask, apache ja saltilla hallinta

Seuraavaksi piti yhdistää Flask ja Apache toimimaan yhdessä. Tätä varten loin uuden tiedoston `trivia.wsgi`, joka on siis toimii ns. sillanrakentaja Apachen ja Flaskin välillä. Loin ensin tiedoston: 

    $ micro trivia.wsgi
    
![wsgi tiedoston sisältö](Pkuvat/wsgi.png)

Seuraavaksi conffi-tiedosto, jotta saadaan Apache toimimaan oikeassa paikassa tätä varten tein uuden kansion ja sen sisälle itse tiedoston: 


    $ mkdir /trivia/apache2
    $ micro trivia.conf

![trivia.conf tiedosto](Pkuvat/trivia.conf.png)

Lopuksi vielä itse `ìnit.sls`, jotta saadaan sovellusten asennus automaattiseksi. Vielä kertaukseksi tänne tulee: 

- Asennetaan Apache2, mod_wsgi ja Flask
- Kopioidaan sovelluksen kaikki tiedostot VM:lle
- Otetaan trivia.conf käyttöön (a2ensite)
- Poistetaan Apachen oletussivusto (a2dissite)
- Varmistetaan että Apache-palvelu on päällä 


Loin `init.sls`- tiedoston `/salt/`-kansion alle: 

        $ cd /srv/salt/
        $ micro init.sls

![init.sls sisältö](Pkuvat/init.sls.png)

Nyt pitäisi olla kaikki palaset valmiina ja päästään testailemaan tätä käytännössä.

## Testailua

Nyt kaikki pitäisi olla sillä mallilla, että ohjelman voi ajaa. Kaikki luomat tiedostot tallentuvat master-koneelle, koska se on synkronoitu vagrantfilessa `salt/`-kansioon. Yritin testata, mutta aluksi Flaskin asennus epäonnistui, joten muokkasin `init.sls`-tiedostoa siten, että se lataa `python3-flask` ja myös apachen default-sivu tuli näkyviin, joten `ìnit.sls`-tiedostoa piti myös muokata siten, että default-sivu laitetaan pois päältä. Unohdin ottaa kuvia näistä virheistä. Lopulta kuitenkin kokeilin ajaa ohjelman ja sain sen toimimaan: 

    $ sudo salt '*' state.apply

![ensimmäinen testiajo](Pkuvat/ekatesti.png)

Sitten siirryin selaimessa `http://192.168.88.10` ja katsoin toimiiko sivu: 

![Nettisivuntesti](Pkuvat/nettisivu.png)

Sivut toimivat oikein hyvin. Ei se kaunis ole, mutta toimii. 

Lopuksi vielä testasin tuhota koneet ja testata toimiiko se myös sen jälkeen. 

    $ vagrant destroy
    $ vagrant up
    $ vagrant ssh master
    # Hyväksytään avain:
    $ sudo salt-key -L
    $ sudo saltkey -A

![saltkey](Pkuvat/saltkey.png)

Seuraavaksi ajo: 

    $ sudo salt '*' state.apply

![toka ajo](Pkuvat/tokaajo.png)

`init.sls`-tiedoston ajo onnistui ja seuraavaksi tarkistin toimiiko osoite: `http://192.168.88.10`, mutta sivu ei kuitenkaan latautunut. Päätin kokeilla vaihtaa slave-koneella portin, jotta se pakottaa IPv4 kuuntelun: 

![portin vaihto](Pkuvat/portinvaihto.png)

ja lopuksi vielä: 

    $ sudo systemctl restart apache2 

Nyt homma taas toimi, mutta en halunnut samaa ongelmaa jatkossa, joten homma piti automatisoida.

Muutin `init.sls`-tiedostoa vielä näin: 

![ipv4 force](Pkuvat/forceipv4.png)

Nyt sivu toimii niin kuin pitää. 

## Kahden koneen peliarkkitehtuuri

Seuraavaksi lähdin muokkailemaan projektia siten, että kaksi konetta (slave1 & slave2) voivat pelata samaan aikaan toisiaan vastaan omista IP-osoitteistaan.

Tässä tavoite: 

| Kone | IP | Toiminta |
|----------|----------|----------|
| Slave1   | 192.168.88.102   | Ajaa trivia-palvelinta   |
| Slave2   | 192.168.88.103 | Ajaa trivia-palvelinta  |

Aloitin muokkaamalla vagrantfileä, jotta saadaa yksi orja lisää: 

    # Master-koneen scripti, jossa asennetaan salt valmiiksi
    $master_script = <<-MASTER_SCRIPT
    set -o verbose
    sudo apt-get update
    sudo apt-get install -y curl tree
    sudo mkdir -p /etc/apt/keyrings
    sudo curl -fsSL https://packages.broadcom.com/artifactory/api/security/keypair/SaltProjectKey/public | sudo tee /etc/apt/keyrings/salt-archive-keyring.pgp
    sudo curl -fsSL https://github.com/saltstack/salt-install-guide/releases/latest/download/salt.sources | sudo tee /etc/apt/sources.list.d/salt.sources
    sudo apt-get update
    sudo apt-get install -y salt-master
    sudo systemctl restart salt-master.service
    MASTER_SCRIPT
    
    # Minion-koneen scripti, jossa asennetaan salt valmiiksi ja otetaan yhteys Master-koneeseen.
    $minion_script = <<-MINION_SCRIPT
    set -o verbose
    sudo apt-get update
    sudo apt-get install -y curl tree
    sudo mkdir -p /etc/apt/keyrings
    sudo curl -fsSL https://packages.broadcom.com/artifactory/api/security/keypair/SaltProjectKey/public | sudo tee /etc/apt/keyrings/salt-archive-keyring.pgp
    sudo curl -fsSL https://github.com/saltstack/salt-install-guide/releases/latest/download/salt.sources | sudo tee /etc/apt/sources.list.d/salt.sources
    sudo apt-get update
    sudo apt-get install -y salt-minion
    echo -e 'master: 192.168.88.101' |sudo tee /etc/salt/minion
    sudo systemctl restart salt-minion.service
    MINION_SCRIPT
    
    Vagrant.configure("2") do |config|
      # Virtuaalikone käyttämään synkronoitua kansiota.
      config.vm.synced_folder ".", "/vagrant", disabled: true
      config.vm.synced_folder "shared/", "/home/vagrant/shared", create: true
      config.vm.box = "debian/bookworm64"
    
      # Master VM
      config.vm.define "master" do |master|
        master.vm.hostname = "master"
        master.vm.network "private_network", ip: "192.168.88.101"
        master.vm.synced_folder "salt/", "/srv/salt", owner: "root", group: "root"
        master.vm.provision "shell", inline: $master_script
    
      master.vm.provider "virtualbox" do |vb|
          vb.memory = 2048
        end
      end
    
      # Slave 1
      config.vm.define "slave" do |slave|
        slave.vm.hostname = "slave"
        slave.vm.network "private_network", ip: "192.168.88.102"
        slave.vm.provision "shell", inline: $minion_script
      end
    
      # Slave 2
      config.vm.define "slave2" do |slave2|
        slave2.vm.hostname = "slave2"
        slave2.vm.network "private_network", ip: "192.168.88.103"
        slave2.vm.provision "shell", inline: $minion_script
      end
    end


Seuraavaksi lähdin muokkaamaan `init.sls`-tiedostoa, jotta se sen voi ajaa sitten, minioneille:

![init muokkaus](Pkuvat/initmuokkaus.png)


Sitten kokeilin ajaa: 

    $ sudo salt '*' state.apply

Toisen minion-koneen osoite ei kuitenkaan toiminut, joten lähdin taas muokkaamaan `init.sls`-tiedostoa: 

![init-tiedosotn muokkaus](Pkuvat/inittoinenmuokkaus.png)

Tämän jälkeen kokeilin uudestaan: 

    $ sudo salt '*' state.apply

Edelleen ensimmäisen minionin osoite toimi hyvin, mutta toisen koneen osoitteessa näkyi apachen default-sivu. Siirryin katsomaan mitä slave2 puolella näkyy: 

    $ ls -l /etc/apache2/sites-enabled/ 

![trivia.conf päällä?](Pkuvat/triviapäällä.png)

Kokeilin: 

    $ sudo systemctl reload apache2

Tällä kertaa sivulla ei näkynyt mitään. Seuraavaksi tarkistin onko `app.py` ja `trivia.wsgi` olemassa ja kyllä ne näyttivät olevan siellä. Kokeilin laittaa sivun pois päältä ja takaisin: 

    $ sudo a2dissite trivia.conf
    $ sudo a2ensite trivia.conf
    $ sudo systemctl restart apache2

Ja vihdoin sain sivun näkyviin myyös toisella slave-koneella. 

![Trivian testaus kahdella koneella](Pkuvat/triviakahdesti.png)

Nyt kun kaikki toimii niin kuin pitää voidaan alkaa laajentamaan peliä. 

## Pelin laajennus

Seuraavaksi loin lisää kysymyksiä peliin muokkaamalla `app.py`-tiedostoa:

![Lisä kysymyksiä app.py](Pkuvat/lisääkysymyksiä.png)

Koodi ei toiminut heti, joten piti tehdä hieman lisää muutoksia `app.py`-tiedostoon, mutta sain lopulta toimimaan. 

Eli nyt slave-koneilla peli toimii omissa IP-osoitteissaan. 

![Peli näkymä kahdessa eri osoitteessa](Pkuvat/elokuvatrivia.png)

![tulosten vertailu](Pkuvat/tulokset.png)

Kokeilin vielä viimeisen kerran poistaa koneet, käynnistää uudestaan ja kokeilla toimiiko homma alusta asti saltin avulla.

    $ vagrant destroy
    $ vagrant up
    $ vagrant ssh master
    # Hyväksytään avaimet
    $ Sudo salt-key -L
    $ Sudo salt-key -A
    $ sudo salt '*' state.apply

Ja sitten oli totuuden hetki toimiiko kaikki odotetusti: 

slave1: 192.168.88.102

slave2: 192.168.88.103

![viimeinen testi](Pkuvat/viimeinentesti.png)
![pelien lopputulokset](Pkuvat/lopputulos.png)

Kaikki näytti toimivan odotetusti!

## Etusivu. Laita projektisi etusivulle tärkeimmät tiedot 

Loin repon, jossa on yksinkertaiset asennusohjeet. Sivu löytyy [täältä](https://github.com/jerebjo/Elokuvatrivia/blob/main/README.md) 

## Ajankäyttö

Hommiin kului aikaa ainakin 10 tuntia.

## Lähteet 

- Karvinen, T. 4.11.2024. Two Machine Virtual Network With Debian 11 Bullseye and Vagrant. Luettavissa: https://terokarvinen.com/2021/two-machine-virtual-network-with-debian-11-bullseye-and-vagrant/ Luettu: 3.5.2025
- jerebjo. 2025. h2 soittokotiin. Luettavissa: https://github.com/jerebjo/Palvelinten-hallinta/blob/main/h2%20soitto%20kotiin.md Luettu: 3.5.2025
- Karvinen, T. 2025. Tehtävänanto. h5 Miniprojekti. Luettavissa: https://terokarvinen.com/palvelinten-hallinta/#h5-miniprojekti Luettu: 3.5.2025
- nurminenkasper. 1.5.2025. h5 Miniprojekti. Luettavissa: https://github.com/nurminenkasper/Palvelinten-Hallinta/blob/main/h5/h5-Miniprojekti.md Luettu: 3.5.2025
- Karvinen, T. 3.4.2024. Hello Salt Infra-as-Code. Luettavissa: https://terokarvinen.com/2024/hello-salt-infra-as-code/ Luettu: 4.5.2025.
- Karvinen,T 28.3.2023. Salt Vagrant - automatically provision one master and two slaves. Luettavissa: https://terokarvinen.com/2023/salt-vagrant/#infra-as-code---your-wishes-as-a-text-file Luettu: 5.5.2025
  

