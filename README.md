# Distribuert kø

**Utviklet på Digipost, basert på Hazelcast.**

### Visning

For å vise presentasjonen må avhengigheter lastes ned med [**Bower**](http://bower.io/). **Bower** kan installeres med [**npm**](https://www.npmjs.com/). **npm** kan installeres med [**Homebrew**](http://brew.sh/). **Homebrew** kan installeres som beskrevet på http://brew.sh hvis du bruker **OS X**. **OS X** kan installeres med... :unamused: :trollface:

Gitt at du allerede har Homebrew installert. I katalogen du har sjekket ut dette git-repoet, kjør følgende kommandoer:

    $ brew install npm
       ...<snip>...
    $ npm install bower
       ...<snip>...
    $ bower install

Den siste kommandoen laster ned avhengigheter til presentasjonen til katalogen `bower_components`.

I tillegg trenger du å serve filene med en Web-server. Det fungerer ikke å åpne `index.html` i en nettleser [direkte fra filsystemet](https://github.com/gnab/remark/wiki#external-markdown).

Jeg anbefaler å installere [**http-server**](https://www.npmjs.com/package/http-server) med **npm**

    $ npm install http-server -g
      ...<snip>...
    $ http-server
    Starting up http-server, serving ./ on: http://0.0.0.0:8080
    Hit CTRL-C to stop the server


---------------------------------------------------

Presentasjonen er laget med [Remark](https://github.com/gnab/remark), og benytter i tillegg [js-sequence-diagrams](https://bramp.github.io/js-sequence-diagrams/) og [yuml.me](http://yuml.me) til å tegne diagrammer.
