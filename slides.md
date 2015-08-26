
class: center, inverse

# .large[Distribuert kø]

--

## &ndash; selvbygget &ndash;



---

# &ndash; Ok, vi trenger å fikse et velkjent problem, og som det finnes fritt tilgjengelige løsninger for.
--

# .right[.red[&ndash; Fett, vi lager det sjæl!]]





---
class: center, middle, inverse

# Problemstilling


---
classdiagram: sdp1

# Sikker digital post (SDP)

<pre class="transform" id="sdp1">
  [Offentlig avsender]-EBMS->[Meldingsformidler{bg:palegreen}]
  [Meldingsformidler]-EBMS->[Sikker digital postkasse{bg:salmon}]
</pre>


???

- overordnet skisse over SDP (sikker digital post)
- all kommunikasjon går via EBMS (meldingsarkitektur styrt av DIFI)
- EBMS: transport-kvittering og applikasjonskvittering.
  - transport: responsen på HTTP-request. indikerer om meldingen er mottatt, og komponenten tar ansvar for videre behandling
  - applikasjon: forretningskvittering som tilgjengeliggjøres når prosessering av en melding er ferdig (ok eller feil).


---
classdiagram: sdp2

# Sikker digital post (SDP)

<pre class="transform" id="sdp2">
  [Offentlig digital post]EBMS->[Sikker digital postkasse{bg:salmon}]
</pre>

???

- slår sammen Avsender og Meldingsformidler til "offentlig post".
- offentlig + postkassa, a.k.a. Digipost

---
classdiagram: sdp3
# Sikker digital post (SDP)

<pre class="transform" id="sdp3">
  [Offentlig digital post]EBMS->[Offentlig adapter{bg:salmon}]
  [Offentlig adapter]-REST->[Digipost{bg:salmon}]
</pre>

???

- implementerer "offentlig sikker digital postkasse" v.h.a. av en adapter-komponent som gjenbruker Digipost sitt opprinnelige REST-API

---
classdiagram: sdp4
# Sikker digital post (SDP)

<pre class="transform" id="sdp4">
  [Offentlig digital post]EBMS->[Offentlig adapter{bg:salmon}]
  [Offentlig adapter]-REST'ish->[Digipost{bg:salmon}]
</pre>

???

- OK, REST'ish.
- APIet brukes allerede av private virksomheter, samt noen offentlige aktører utenfor SDP.

--

Designprinsipp:

- .red[Offentlig adapter] oversetter fra SDP til Digipost sitt vanlige REST-API for avsendere.

- .red[Digipost] lykkelig uvitende om EBMS og annen arkitektur i SDP.

???

- adapter oversetter
- Digipost "beskyttet" mot EBMS

--

- kommunikasjon mellom Offentlig adapter og Digipost gjøres .red[synkront].

???

- offentlig adapter "jukser" altså.
- Strategi: ta imot EBMS-melding i adapter, oversette, og sende videre til Digipost, få OK-respons fra Digipost tar så kort tid at vi sender _både_ transport-kvittering og applikasjonskvittering tilbake til Meldingsformidler når vi er ferdige med å videresende meldingen til Digipost sitt REST-API.






---

class: center, inverse

# Og dette fungerer fint, fordi:

## .red[nettverkskommunikasjon er pålitelig]

--

background-image: url(failcat.jpg)


# .stamp[.huge[\#FAIL]]

.footnote[[wikipedia.org/wiki/Fallacies_of_distributed_computing](https://en.wikipedia.org/wiki/Fallacies_of_distributed_computing)]

???

Nei, det gjør ikke det.





---
classdiagram: sdp5
# Sikker digital post (SDP)

<pre class="transform" id="sdp5">
  [Offentlig digital post]EBMS->[Offentlig adapter{bg:salmon}]
  [Offentlig adapter]-REST'ish->[Digipost{bg:salmon}]
</pre>

--

- Hva skjer hvis (når) vi opplever et .red[brudd i kommunikasjonen] mellom .bold[Offentlig adapter] og Digipost?

???

- Requesten blir avbrutt, og det kastes exception i Adapteret.
- Adapter sender feilkvittering tilbake til Meldingsformidler
- feilkvittering tilgjengeliggjøres til avsender
- AVSENDER OPPLEVER FEIL I EN SABLA BRA LØSNING!

## Løsning

- Implementere en retrymekanisme i adapter
  - feilende melding persisteres i database, og det sendes "transport OK" tilbake til Meldingsformidler
  -

--

- Hva skjer når Digipost bruker for lang tid på å prosessere en melding slik at Offentlig adapter opplever en .red[READ TIMEOUT] mot Digipost?

???

- Igjen, requesten blir avbrutt, og det kastes exception i Adapteret.
- Feilkvittering går tilbake til avsender.
- BREVET HAR LIKEVEL BLITT SENDT, SELV OM AVSENDER HAR FÅTT BESKJED OM AT DET FEILET!

## Løsning

- Digipost kalkulerer et fingerprint av relevante immutable meldingsdata
- dersom det sendes en melding med samme ID og beregnet fingerprint som en som allerede er sendt, gis det "OK"-svar tilbake til Adapter uten noe mer prosessering i Digipost
- Adapter pusher leveringskvittering til Meldingsformidler, som tilgjengeliggjør den for avsender.



---

class: center, inverse

# .large[Retry queue]


## Distribuert kø

--

### .red[`java.util.concurrent`] (Java 8)

--

### Et par distribuerte datastrukturer i .red[Hazelcast].

--

### Vanlig .red[relasjonsdatabase] for persistens.








---

layout: true
name: retryqueue-design
class: center

# Retry queue &ndash; design


---
template: retryqueue-design
sequencediagram: dia1


???

Designet av retrymekanismen.

--

<div id="dia1">
  participant Meldingsformidler as MF
  participant Adapter as A
  participant Digipost as DP
  MF->A: SDP-melding  
  A->DP: melding
  DP->A: ERROR
</div>

???

Melding sendes fra Adapter til Digipost
og feiler.



---
template: retryqueue-design
sequencediagram: dia2

<div id="dia2">
  participant Meldingsformidler as MF
  participant Adapter as A
  participant Digipost as DP
  MF->A: SDP-melding
  A->DP: melding
  DP->A: ERROR
  note over A: persister melding
  A->MF: OK transportkvittering
</div>

???

- Adapter persisterer feilende melding i en "kø-tabell" i databasen.
- Sender OK Transportkvittering til Meldingsformidler.
- Meldingsformidler venter nå på applikasjonskvittering fra postkassen (altså adapter)






---
template: retryqueue-design
sequencediagram: dia3

<div id="dia3">
  participant Meldingsformidler as MF
  participant Adapter as A
  participant Digipost as DP
  MF->A: SDP-melding
  A->DP: melding
  DP->A: ERROR
  note over A: persister melding
  A->MF: OK transportkvittering

  note over A: plukker opp melding til retry
  A->DP: melding
  DP->A: OK
  A->MF: leveringskvittering
</div>

???

- Persistert melding plukkes opp av én av nodene (distribuert lås, Hazelcast)
- Legges på distribuert kø
- Alle Adapter-noder plukker fortløpende meldinger fra distribuert kø og forsøker å sende de på nytt til Digipost
- Når meldinger sendes OK, går det leveringskvittering tilbake til Meldingsformidler, som tilgjengeliggjør denne for avsender.






---

template: retryqueue-design
sequencediagram: dia4

<div id="dia4">
  participant DB
  participant Populator as POP
  participant WorkQueue as Q
  note over POP: tryLock()
  DB->POP: melding
  POP-->>Q: exists?
  POP->Q: melding
</div>

???

- Populatoren spør om den kan få distribuert lås
- drar opp meldinger fra DB
- pusher de meldingene som ikke finnes i køen fra før






---

template: retryqueue-design
classdiagram: queueclasses1

<pre class="transform" id="queueclasses1">
  [WorkQueue|push(){bg:gold}]
  [WorkQueue]++->[PendingWork|push();poll()]
  [WorkQueue]++->[CurrentlyBeingProcessed|put();discard();isProcessing()]  
</pre>

???

En WorkQueue består av to datastrukturer:
- Pending work
- Currently being processed






---

template: retryqueue-design
classdiagram: queueclasses2

<pre class="transform" id="queueclasses2">
  [WorkQueue|push(){bg:gold}]
  [WorkQueue]++->[PendingWork|push();poll()]
  [WorkQueue]++->[CurrentlyBeingProcessed|put();discard();isProcessing()]  
  [CurrentlyBeingProcessed]^-[HazelcastCurrentlyBeingProcessed|inProcess: com.hazelcast.core.ISet{bg:dodgerblue}]
  [PendingWork]^-[HazelcastPendingWork|pending: com.hazelcast.core.IQueue{bg:dodgerblue}]
</pre>

???

Disse to datastrukturene er implementert av Hazelcast:
- pending work er en distribuert Queue
- Currently being processed er et distribuert Set
- Alle nodene ser det samme innholder i disse strukturene.
- Hazelcast sikrer at nodene plukker hver sine elementer fra køen.






---
template: retryqueue-design
sequencediagram: dia5

<div id="dia5">
  participant WorkQueue as Q
  participant WorkerPool as WRK
  participant Poller as POLL
  participant ResendMelding as TASK
  WRK->POLL: hent resend-melding-worker
  Q->POLL: hent melding
  POLL->TASK: worker.work(melding)
  POLL->POLL: gjenta
</div>

???

Poller går i en evig loop
- først henter tilgjengelig worker
- så henter melding
- til slutt kaller `worker.work(melding)` (async, returnerer umiddelbart)
- poller gjentar






---
template: retryqueue-design
sequencediagram: dia6

<div id="dia6">
  participant DB
  participant WorkQueue as Q
  participant WorkerPool as WRK
  participant Poller as POLL
  participant ResendMelding as TASK
  WRK->POLL: hent worker
  Q->POLL: hent melding
  POLL->TASK: worker.work(melding)
  POLL->POLL: gjenta
  note over TASK: sendt OK
  TASK->DB: slett melding
</div>

???

- tasken ResendMelding startes i egen tråd.
- Forsøker å sende meldingen til Digipost
- Oppdaterer database i henhold til om den fullfører OK eller feiler.





---
template: retryqueue-design
sequencediagram: dia7

<div id="dia7">
  participant Populator as POP
  participant WorkQueue as Q
  participant WorkerPool as WRK
  participant Poller as POLL
  participant ResendMelding as TASK
  WRK->POLL: hent worker
  Q->POLL: hent melding
  POLL->TASK: worker.work(melding)
  POLL->POLL: gjenta
  note over Q: få elementer?
  Q->POP: "snart tomt!"
  note over POP: restarter seg selv
</div>

???

Straks populatoren har funnet meldinger til resending i databasen og populert køen, så går den i "sovemodus".
Når antall meldinger i køen går under en satt terskelverdi, vil køen signalisere til populatoren at det er greit å forsøke å pushe flere meldinger. Populatoren vil da følgelig se i databasen igjen.








---
layout: false
class: center, inverse, middle

.large[show me teh]
.huge[.red[codez]]

???

Poller: (enklest)
- vise gangen
- vise `OneTimeToggle`: en spesialisert boolean som bare går an å switche én gang. Brukes til å kontrollere livssyklusen til komponenten. Kan kun startes en gang, og brukes til kontrollert shutdown.

Populator: (litt mer avansert)
- loop som gjentas helt til den finner elementer å pushe på køen
- distribuert lås -> kun 1 node spør databasen om gangen.
- loopen restartes først når den får signal om det
  - trådsikkert
  - flere kan si fra samtidig
  - kan si fra når loopen allerede kjører
  - `SignalMediator`

CompletableFuture vs Future
- CompletableFuture kan ikke .cancel()-eres
- http://www.nurkiewicz.com/2015/03/completablefuture-cant-be-interrupted.html
- CompletableFuture brukes til små oppgaver som forkes. (f.eks. worker.work())
- Future brukes som "handle" for bakgrunnsprosesser. Kaller .cancel() for å avslutte prosessen.

Testing:
- PopulatorTest
  - regularPopulateWhenDbIsEmpty
  - whenQueueIsUnderThresholdNewPopulateIsCalled
- FullPopulatorQueueWorkerPipelingTest
  - flere lag med asynkronitet
  - ingen bruk av Thread.sleep() for å vente til asynkrone tasks "helt sikker" har blitt kjørt.
- Mye testing av trådsikkerhet
- Testdekning av trådkritiske komponenter (Populator, Poller, WorkQueue) ~90%











---
layout: false
class: center, inverse, middle

# Takk for meg!


.left[.footnote[
  Laget med [github.com/gnab/remark](https://github.com/gnab/remark)

  Bokser og piler laget med [yuml.me](http://yuml.me/)

  Sekvensdiagrammer laget med [bramp.github.io/js-sequence-diagrams/](https://bramp.github.io/js-sequence-diagrams/)
]]

???


### Anbefalt lesing:

http://www.nurkiewicz.com/2015/03/completablefuture-cant-be-interrupted.html

http://docs.oracle.com/javase/8/docs/api/java/util/concurrent/package-summary.html



---

class: center, middle, inverse

Page intentionally left blank

???
```
retrymelding.in-process   0
retrymelding.pending      0
retrymelding.processed.last-Days  0
retrymelding.processed.last-Hours 0
retrymelding.workers.available    9
retrymelding.workers.processing-rate     Infinity per Minutes
retrymelding.workers.total     10
```
