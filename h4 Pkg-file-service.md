# h4 Pkg-file-service 

## a) Apache easy mode. Asenna Apache, korvaa sen testisivu ja varmista, että demoni käynnistyy. 

Aloitin tehtävän ssh-kirjautumalla viime tehtävistä valmiiksi tehdylle master-koneelle:

Koneet päälle: 

    $ vagrant up

ssh-kirjautuminen: 

    $ vagrant ssh t001

Asensin apache2-ohjelman aluksi käsin: 

    $ sudo apt-get -y install apache2

Seuraavaksi kokeilin, että testisivu on päällä: 

    $ curl localhost

![Testisivun testaus](Kuvat/testisivu.png)

Sitten korvasin, testisivun omalla sivulla. 

    $ echo "<h1>Tämä on testi</h1>" | sudo tee /var/www/html/index.html
    $ curl localhost

![Testisivun korvaus](Kuvat/omasivu.png)

Nyt on tehty käsin niin seuraavaksi tehdään automatisoidusti. 

Eli loin uuden moduulin `apache`: 

    $ sudo mkdir -p /srv/salt/apache/

Ja sinne loin uuden `.sls`-tiedoston microlla: 

    $ micro apachesetti.sls

Sitten muokkasin .sls-tiedoston seuraavanlaiseksi: 

![sls-tiedoston muokkaus](Kuvat/apachesetti.png)

Seuraavaksi yritin ajaa tiedostoa: 

    $ sudo salt '*' state.apply apache.apachesetti

Mutta sain kuitenkin seuraavan virheviestin: 

![Erromessage](Kuvat/vika.png)

Korjasin virheen poistamalla kaksoispisteen `pkg.installed` perästä: 

![Viankorjaus](Kuvat/viankorjaus.png)

Sitten kokeilin ajaa uudestaa: 

![Testiajo](Kuvat/testiajo.png)

Se näytti toimivan hyvin. Kokelin vielä toimiiko hommat minion-koneella: 

![minion-koneen curl-testi](Kuvat/miniontesti.png)
