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
