---
title: Rezervační systém
type: docs
prev: docs/folder/
---

## Struktura sedadel

Každý spoj tvoří jeden až n vozů. Vozy obsahují contingenty, což je abstraktní blok sedadel nebo jiných prvků. Typ contingentu určuje, jestli se jedná o povinně místenkový oddíl nebo kapacitu. Kapacita se používá pro limitované zdroje, jako jsou zvířata, jízdní kola apod., kterých lze do vozu prodat pouze omezené množství, ale které nemají konkrétní místo.

Vůz je pak značen pořadím v soupravě a popisem. 

Spoje jezdí po trase, která je určena číslem spoje (*LineNumber*), např. R1322 je vlak na trase Liberec - Ústí nad Labem. Na této trase jezdí vlak několikrát denně, každý spoj v rámci dne má vlastní LineNumber (a může jet po jiné trase). Na druhou stranu linka 1322 jezdí denně a pro každou jízdu existuje záznam v tabulce **LineInstance**, to je konkrétní jízda, na kterou zajišťujeme místenky. Pokud linka v některý den nejede, jednoduše chybí záznam v tabulce LineInstance se dnem, kdy spoj nejezdí.

## Rezervace místenky

### Získání vstupních dat

Pro úspěšnou rezervaci místa je třeba znát linku, stanice z a do a datum jízdy. 
Pro stanice využijeme číselník, který poskytuje volání ```/v1/Constants/GetAllPlaces```. Seznam stanic je obsáhlý a je třeba si ho uložit lokálně, opakované volání by příliš vytěžovalo komunikaci.
Dle číselníku přiřadíme stanicím z a do konkrétní identifikátory. Nyní se volá ```/v1/Route/ExtSearch```, čímž se získá číslo linky; v případě přestupů více linek.

Řekněme, že chceme jet 31.1.2024 v 6 hodin z Liberce do České Lípy. Liberec má ID=1453, Česká Lípa má ID=2187. Vyhledávání vrátí linku "R 1322". kde "1322" je číslo linky. 

## Zjištění volné kapacity

V této chvíli je třeba zjistit, jestli dotyčný spoj má volnou kapacitu. K tomu je třeba zavolat ```/v1/Reservation/ExtGetFreeSeatsCount``` s následujícími parametry:
```json
{
  "lineNumber": "1322",
  "date": "2024-01-31T06:00:00.000Z",
  "traceStationFromId": 1453,
  "traceStationToId": 2187
}
```
Výsledek bude přibližně tento:
```json
[
  {
    "contingentTypeId": 1,
    "capacityResult": {
      "capacity": 63,
      "available": 54
    }
  }  ,
   {
    "contingentTypeId": 3,
    "capacityResult": {
      "capacity": 10,
      "available": 8
    }
  },
   {
    "contingentTypeId": 4,
    "capacityResult": {
      "capacity": 20,
      "available": 17
    }
  }
]
```

Znamená to, že tento spoj má jeden contingent typu 1, což odpovídá povinně místenkovému vozu. Jeho celková kapacita je 63 míst a zbývá 54 míst. Dále je možné prodat celkem 10 jízdenek pro jízdní kolo (TypeId=3), přičemž k dispozici je ještě 8 a 20 jízdenek pro psa (TypeId=4), kde je k dispozici 17 míst.

## Rezervace místenky

Řekněme, že potřebujeme jednu místenku, zavoláme tedy ```/v1/Reservation/ExtMakeReservations```
s parametry:
```json
{
  "lineNumber": "1322",
  "date": "2023-12-05T00:00:00.000Z",
  "traceStationFromId": 1453,
  "traceStationToId": 2187,
  "count": 1
}
```
a dostaneme odpověď (vrací se pole pro případ, že bychom zadali víc než jednu místenku):
```json
[  
  {
    "seat": {
      "isAvailable": false,
      "seatAttributes": [
        "Window",
        "V230",
        "Silent"
      ],
      "isEnabled": false,
      "description": "11",
      "orientation": "Backward",
      "coordX": 0,
      "coordY": 4,
      "floor": 1,
      "vehicle": {
        "model": "845.P",
        "sequence": 1,
        "description": "1",
        "vehicleAttributes": [
          "WiFi",
          "Silent",
          "V230"
        ],
        "seatMap": [],
        "contingents": [],
        "shiftingId": "29160129-c223-4022-b186-fba25b085377"
      }
    },
    "externalId": "193d04af-9dc8-4767-ad6f-9115886a85e6",
    "traceFromId": 1453,
    "traceFromName": "Liberec",
    "traceToId": 2187,
    "traceFromSequence": 1,
    "traceToName": "Česká Lípa hl.n.",
    "traceToSequence": 4,
    "expiresAt": "2024-01-10T20:25:26.543058+01:00",
    "date": "2023-12-05T00:00:00",
    "lineNumber": "R1322",
    "routeName": "R14B",
    "reservationStatus": "Temporary",
    "vehicleOrder": 1,
    "reservationType": "Reservation"
  }
]
```

#### Popis nejdůležitějších hodnot:

**seat** popisuje vybrané sedadlo, a to jeho pozici ve voze (coordX, coordY), jeho vlastnosti (seatAttributes), 
  
**vehicle** informace o vozu. Model je typ vpzu. důležité parametry jsou *sequence* (na jaké pozici je vůz řazen) a *description*, to je označení vozu pro cestující. Pokud se změní řazení vozu v soupravě, např. mezi dva vozy se zařadí další, změní se hodnota *sequence*, ale nesmí se změnit *description*, protože pak by cestující ztratili orientační bod.

**shiftingId** je identifikátor řazení vozu. Jak funguje řazení? V systému máme omezený počet vozů, je to vlastně šablona, popisující jak vůz vypadá, kde má které sedadlo apod. Pokud ho chci zařadit do soupravy, vytvořím záznam o řazení (*shifting*), který propojuje šablonu vozu s konkrétní jízdou (*lineInstance*) a pozicí v soupravě (*sequence*). 

**externalId** je hodnota, pomocí které je místenka identifikována pro budoucí operace.
Následují informace o trase (traceFromId, traceFromName...), **date** určuje den jízdy a **lineNumber** číslo linky.

**reservationStatus** Temporary určuje, že se jedná o dočasnou rezervaci, která má nastavenou automatickou expiraci (**expiresAt**). Po tomto čase přestane místenka existovat, pokud není potvrzena. 

## Potvrzení rezervace

K tomu je třeba zavolat ```/v1/Reservation/ExtConfirmReservations``` s parametrem:

```json
{
  "externalReservationsIds": [
    "193d04af-9dc8-4767-ad6f-9115886a85e6"
  ]
}
```

Vrátí se opět informace o místence:
```json
[
  {
    "seat": {
      "isAvailable": false,
      "seatAttributes": [
        "Window",
        "V230",
        "Silent"
      ],
      "isEnabled": false,
      "description": "11",
      "orientation": "Backward",
      "coordX": 0,
      "coordY": 4,
      "floor": 1,
      "vehicle": {
        "model": "845.P",
        "sequence": 1,
        "description": "1",
        "vehicleAttributes": [],
        "seatMap": [],
        "contingents": [],
        "shiftingId": "29160129-c223-4022-b186-fba25b085377"
      }
    },
    "externalId": "193d04af-9dc8-4767-ad6f-9115886a85e6",
    "traceFromId": 1453,
    "traceFromName": "Liberec",
    "traceToId": 2187,
    "traceFromSequence": 1,
    "traceToName": "Česká Lípa hl.n.",
    "traceToSequence": 4,
    "expiresAt": null,
    "date": "2023-12-05T00:00:00",
    "lineNumber": "R1322",
    "routeName": "R14B",
    "reservationStatus": "Confirmed",
    "vehicleOrder": 1,
    "reservationType": "Reservation"
  }
]
```
Nyní je již **reservationStatus** nastaven na Confirmed a hodnota **expiresAt** je *NULL*.

## Uvolnění dočasné rezervace

V případě, že se nákup ruší bez potvrzení, volá se místo potvrzení ```/v1/Reservation/ExtReleaseReservations``` s parametrem:
```json
{
  "externalReservationsIds": [
    "193d04af-9dc8-4767-ad6f-9115886a85e6"
  ]
}
```
která vrací pouze *true/false*. Tím se rezervovaná místenka uvolní pro ostatní zájemce a není třeba čekat na vypršení expirace.

# Změna sedadla

Pokud nevyhovuje automaticky přidělené sedadlo, je možné jej změnit. Nejprve se zavolá  ```/v1/Reservation/ExtGetVehicles``` s parametrem:
```json
{
  "lineNumber": "1322",
  "date": "2023-12-05T00:00:00.000Z",
  "traceStationFromId": 1453,
  "traceStationToId": 2187 
}
```

Jako odpověď dostaneme seznam řazených vozů, pasivních prvků (item) a sedadel (seat)
```json
  [
  {
    "model": "845.P",
    "sequence": 1,
    "description": "1",
    "vehicleAttributes": [],
    "seatMap": [
      {
        "description": "Východ",
        "orientation": "Backward",
        "externalId": "8be67ae9-0d6e-45c9-ad12-2ba24e7433e9",
        "coordX": 0,
        "coordY": 0,
        "floor": 1,
        "availability": "Unknown",
        "itemType": "Exit",
        "seatAttributes": []
      },
      {
        "description": "Prázdný prostor",
        "orientation": "Backward",
        "externalId": "5fc6c8ab-edca-4b9f-9e68-a0f947b3050e",
        "coordX": 1,
        "coordY": 0,
        "floor": 1,
        "availability": "Unknown",
        "itemType": "Empty",
        "seatAttributes": []
      },
	  /*  .... */
      {
        "description": "11",
        "orientation": "Backward",
        "externalId": "2bb66e74-d0de-429a-9d78-7ad01108992b",
        "coordX": 0,
        "coordY": 4,
        "floor": 1,
        "availability": "Sold",
        "itemType": "Seat",
        "seatAttributes": []
      },
      {
        "description": "13",
        "orientation": "Backward",
        "externalId": "70123deb-4e70-4ab3-a4ab-6f50da18c780",
        "coordX": 1,
        "coordY": 4,
        "floor": 1,
        "availability": "Available",
        "itemType": "Seat",
        "seatAttributes": []
      },
      {
        "description": "Ulička",
        "orientation": "Backward",
        "externalId": "2acf9a37-0e3e-43f7-a82f-6d9718931aa5",
        "coordX": 2,
        "coordY": 4,
        "floor": 1,
        "availability": "Unknown",
        "itemType": "Corridor",
        "seatAttributes": []
      },
      {
        "description": "17",
        "orientation": "Backward",
        "externalId": "394c4cfa-7de5-4edc-b890-94b9dc64428e",
        "coordX": 3,
        "coordY": 4,
        "floor": 1,
        "availability": "Available",
        "itemType": "Seat",
        "seatAttributes": []
      },
      {
        "description": "15",
        "orientation": "Backward",
        "externalId": "766cc612-0185-4ca5-9613-031d74d13fe0",
        "coordX": 4,
        "coordY": 4,
        "floor": 1,
        "availability": "Available",
        "itemType": "Seat",
        "seatAttributes": []
      },
      /*  .... */
    ],
    "contingents": [
      {
        "capacity": 39,
        "available": 38,
        "contingentType": "Reservation"
      },
      {
        "capacity": 24,
        "available": 24,
        "contingentType": "Reservation"
      },
      {
        "capacity": 10,
        "available": 10,
        "contingentType": "Bicycle"
      },
      {
        "capacity": 0,
        "available": 0,
        "contingentType": "Disabled"
      }
    ],
    "shiftingId": "29160129-c223-4022-b186-fba25b085377"
  }
]
```
Data jsou zkrácena. Důležité informace jsou v sekci **contingens**, které popisují, jaké typy rezervací vůz nabízí (místenka, zvíře, kolo, invalidé...). 

Ve struktuře **seatMap** je popsán vůz včetně souřadnic jednotlivých pasivních prvků (stolek, ulička...) a především sedadel. Zde je důležitá vlastnost *availability*, která popisuje, je-li možné sedadlo obsadit.

Voláním  ```/v1/Reservation/ChangeReservation``` s parametry 
```json
{
  "reservationExternalId": "193d04af-9dc8-4767-ad6f-9115886a85e6",
  "seatExternalId": "766cc612-0185-4ca5-9613-031d74d13fe0",
  "shiftingExternalId": "29160129-c223-4022-b186-fba25b085377"
}
```
kde **reservationExternalId** je Id rezervace, **seatExternalId** je Id požadovaného sedadla a **shiftingExternalId** je identifikátor vozu a jeho řazení, lze změnit místo přiřazené rezervaci. Lze měnit místa pouze u rezervací ve stavu Temporary, po potvrzení je již rezervace neměnná.

Funkce vrací **novou rezervaci**. Původní rezervace je automaticky uvolněna, ale je odpovědnosti klienta, aby identifikátor rezervace změnil na nově vrácený identifikátor. Při ukončení nákupu je třeba zavolat ConfirmReservations s tímto novým Id!
```json
[
  {
    "seat": {
      "isAvailable": false,
      "seatAttributes": [],
      "isEnabled": false,
      "description": "15",
      "orientation": "Backward",
      "coordX": 4,
      "coordY": 4,
      "floor": 1,
      "vehicle": {
        "model": "845.P",
        "sequence": 1,
        "description": "1",
        "vehicleAttributes": [],
        "seatMap": [],
        "contingents": [],
        "shiftingId": "29160129-c223-4022-b186-fba25b085377"
      }
    },
    "externalId": "e3e7abb2-15b1-4b75-bd9e-215eb8d49350",
    "traceFromId": 1453,
    "traceFromName": "Liberec",
    "traceToId": 2187,
    "traceFromSequence": 1,
    "traceToName": "Česká Lípa hl.n.",
    "traceToSequence": 4,
    "expiresAt": "2024-01-10T21:00:00",
    "date": "2023-12-05T00:00:00",
    "lineNumber": "R1322",
    "routeName": "R14B",
    "reservationStatus": "Temporary",
    "vehicleOrder": 1,
    "reservationType": "Reservation"
  }
]
```