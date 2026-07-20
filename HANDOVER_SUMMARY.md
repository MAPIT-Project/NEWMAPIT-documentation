# Zakelijke overdrachtssamenvatting MAPIT

**Statusdatum:** 20 juli 2026

**Advies:** geschikt voor overdracht als ontwikkelproject, **niet vrijgeven als productieklare dienst**

## Projectbeschrijving

MAPIT is een digitale assistent voor het vinden en beoordelen van Europese subsidiekansen en voor
ondersteuning bij het schrijven van voorstellen. Het product bestaat uit een webapplicatie en een
AI-ondersteunde achterliggende dienst. Horizon Europe is het verst uitgewerkt; de dekking en
validatie van andere Europese programma's is beperkter.

## Overdrachtstatus

De documentatie is na de review aanzienlijk beter overdraagbaar: er is nu één startpunt, een
gecontroleerd overzicht van de interface tussen voor- en achterkant, en centrale informatie over
beveiliging, deployment, beheer, gegevens en bekende risico's. Ook zijn automatische controles
voor beide applicatiedelen toegevoegd en is een blokkade in de verzameling van backendtests
opgelost.

De oorspronkelijke [documentatiereview](DOCUMENTATION_REVIEW.md) gaf een readinessscore van
**37/100**. Dit was een nulmeting vóór de verbeteringen en is niet formeel opnieuw gescoord. De
documentatie-acties met prioriteit P0 zijn uitgevoerd, maar de onderliggende productrisico's zijn
niet door documentatie alleen opgelost. De laatst aanwezige functionele kwaliteitsmeting blijft
**23 van 42 scenario's geslaagd (54,8%)**, tegenover een vereiste 80%. De actuele conclusie is
daarom: overdraagbaar voor verdere ontwikkeling en een gecontroleerde demonstratieomgeving, maar
niet gereed voor onbeheerde productie of een zelfstandige release.

## Vijf belangrijkste risico's

1. **Onvoldoende bewezen productkwaliteit.** Negentien van de 42 kernscenario's falen in de
   laatste finale meting. Belangrijke antwoorden en gespreksstromen moeten vóór vrijgave opnieuw
   worden getest en verbeterd (`QUAL-001`).
2. **Een complete gegevensomgeving is niet reproduceerbaar.** De representatieve database en de
   benodigde ruwe gegevens worden niet volledig met het project meegeleverd. Een nieuw team kan
   daardoor niet zelfstandig dezelfde omgeving opbouwen (`DATA-001`, `DATA-002`).
3. **Toegangsafscherming is niet overal afgedwongen.** Gebruikersidentiteit en toegang tot gedeelde
   of persoonlijke gegevens worden in delen van de keten onvoldoende gecontroleerd. Dit kan tot
   ongewenste inzage of wijziging leiden (`API-001`, `SEC-006`, `SEC-007`).
4. **Beveiligingsinstellingen vragen vóór productie om herstel.** Er zijn risico's rond een
   standaard geheime sleutel, sessiecookies, sleutelbenaming en uitgeschakelde bescherming tegen
   overmatig gebruik (`SEC-002` t/m `SEC-005`).
5. **Continuïteit en beheer zijn nog niet ingericht.** Back-ups, hersteltests, waarschuwingen,
   verantwoordelijkheden en automatische gegevensverversing zijn beschreven maar nog niet
   operationeel aangetoond (`OPS-001`, `OPS-002`).

De volledige onderbouwing en workarounds staan in [Bekende problemen en risico's](KNOWN_ISSUES_AND_RISKS.md).

## Uitgevoerd tijdens de review

- Een volledige documentatiereview met score, tegenstrijdigheden en geprioriteerde acties.
- Negen nieuwe of volledig herziene P0-documenten voor onboarding, gegevens, API, database,
  beveiliging, deployment, beheer en risico's; acht misleidende documenten zijn als archief
  gemarkeerd.
- Automatische controles voor backend en frontend, plus herstel van de backendtestverzameling en
  een vastgelegde frontendfoutenbaseline.
- Een migratie-overzicht met bestanden, controlesommen, afhankelijkheden en bekende hiaten.
- In Wave 3: complete overzichten van omgevingsvariabelen, tools en afhankelijkheden, deze
  zakelijke samenvatting en een vernieuwde documentatie-index.

## Volgende stappen voor de ontvanger

1. Wijs één producteigenaar en per applicatiedeel een technisch verantwoordelijke aan.
2. Accepteer de huidige status formeel als ontwikkelproject en plan geen productierelease zolang
   de kritieke en hoge risico's niet aantoonbaar zijn gesloten.
3. Laat de vijf risico's hierboven oplossen, met eigenaar, einddatum en controlebewijs per item.
4. Lever een goedgekeurde, volledige gegevensset met versie, herkomst en controlegetal, en test de
   opbouw van een volledig nieuwe omgeving.
5. Maak alle automatische kwaliteitscontroles groen en blokkerend; herhaal daarna de 42
   kernscenario's en leg het releasebesluit vast.
6. Voer vóór ingebruikname een onafhankelijke beveiligingscontrole en een back-up-/hersteltest uit.

## Contact voor vragen

Technische overdrachtsvragen: **Anmar Ram** — [anmar.noah@gmail.com](mailto:anmar.noah@gmail.com).

De organisatie moet nog een blijvende producteigenaar, beheercontact en escalatiecontact aanwijzen.
