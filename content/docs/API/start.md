---
title: Reservation system
type: docs
prev: docs/folder/
---

## Seating Structure

Each train consists of one to n vehicles. Vehicles contain contingents, which are abstract blocks of seats or other elements. The type of contingent determines whether it is a mandatory reservation section or a capacity section. Capacity is used for limited resources such as animals, bicycles, etc., which can only be sold in a limited quantity within the vehicles but do not have specific seats.

The vehicles are then identified by their order in the train and description.

## Seat Reservation

### Obtaining Input Data

To successfully reserve a seat, you need to know the route, departure and arrival stations, and the travel date. For stations, we will use a list provided by the call `/v1/Constants/GetAllPlaces`. The station list is comprehensive and should be saved locally to avoid excessive communication. Based on the directory, we assign specific identifiers to departure and arrival stations. Now, we call `/v1/Route/ExtSearch` to retrieve the line number or the line numbers in the case of transfers.

Let's say we want to travel on December 5, 2023, at 16:00 from Liberec to Turnov. Liberec has ID=1453, and Turnov has ID=1482. The search will return the line "R 1073," where 1073 is the line number.

## Checking Available Capacity

At this point, we need to determine if the selected train has available capacity. To do this, we call `/v1/Reservation/ExtGetFreeSeatsCount` with the following parameters:

```json
{
  "lineNumber": "1073",
  "date": "2023-12-05T14:53:20.780Z",
  "traceStationFromId": 1453,
  "traceStationToId": 1482
}
```

The result will be approximately as follows:

```json
[
  {
    "contingentTypeId": 1,
    "capacityResult": {
      "capacity": 63,
      "available": 54
    }
  }
]
```

This means that this train has one contingent of type 1, which corresponds to a mandatory reservation car. Its total capacity is 63 seats, and there are 54 seats available.

## Seat Reservation

Suppose we need one seat; we will call `/v1/Reservation/ExtMakeReservations` with parameters:

```json
{
  "lineNumber": "1073",
  "date": "2023-12-05T00:00:00.000Z",
  "traceStationFromId": 1453,
  "traceStationToId": 1482,
  "count": 1
}
```

and receive a response:

```json
[
  {
    "seat": {
      "isAvailable": false,
      "seatAttributes": ["Window", "V230", "Silent"],
      "isEnabled": true,
      "description": "11",
      "orientation": "Backward",
      "coordX": 0,
      "coordY": 4,
      "floor": 1,
      "vehicle": {
        "model": "845.P",
        "sequence": 1,
        "description": "845-P-L",
        "vehicleAttributes": ["WiFi", "Silent", "V230"],
        "seatMap": [],
        "contingents": []
      }
    },
    "externalId": "f6066a0f-c9c8-4310-ab8f-8180e525acbc",
    "traceFromId": 1453,
    "traceFromName": "Liberec",
    "traceToId": 1482,
    "traceFromSequence": 1,
    "traceToName": "Turnov",
    "traceToSequence": 3,
    "expiresAt": "2023-11-15T15:56:03.759+01:00",
    "date": "2023-12-05T00:00:00",
    "lineNumber": "R1073",
    "routeName": "R14A",
    "reservationStatus": "Temporary",
    "vehicleOrder": 0,
    "reservationType": "Reservation"
  }
]
```

#### Description of key values:

**seat** describes the selected seat, including its position in the car (coordX, coordY), its attributes (seatAttributes), and information about the vehicle.
**externalId** is the value used to identify the reservation for future operations.
Next are route information (traceFromId, traceFromName...), **date** specifies the travel date, and **lineNumber** is the line number.
**reservationStatus** Temporary indicates that it is a temporary reservation with automatic expiration set (**expiresAt**). After this time, the reservation will cease to exist unless confirmed.

## Confirming Reservation

To confirm the reservation, you need to call `/v1/Reservation/ExtConfirmReservations` with the parameter:

```json
{
  "externalReservationsIds": ["f6066a0f-c9c8-4310-ab8f-8180e525acbc"]
}
```

The response will provide information about the reservation again:

```json
[
  {
    "seat": {
      "isAvailable": false,
      "seatAttributes": ["Window", "V230", "Silent"],
      "isEnabled": false,
      "description": "11",
      "orientation": "Backward",
      "coordX": 0,
      "coordY": 4,
      "floor": 1,
      "vehicle": {
        "model": "845.P",
        "sequence": 1,
        "description": "845-P-L",
        "vehicleAttributes": [],
        "seatMap": [],
        "contingents": []
      }
    },
    "externalId": "f6066a0f-c9c8-4310-ab8f-8180e525acbc",
    "traceFromId": 1453,
    "traceFromName": "Liberec",
    "traceToId": 1482,
    "traceFromSequence": 1,
    "traceToName": "Turnov",
    "traceToSequence": 3,
    "expiresAt": null,
    "date": "2023-12-05T00:00:00",
    "lineNumber": "R1073",
    "routeName": "R14A",
    "reservationStatus": "Confirmed",
    "vehicleOrder": 0,
    "reservationType": "Reservation"
  }
]
```

Now, the **reservationStatus** is set to Confirmed, and the value of **expiresAt** is _NULL_.

## Releasing Temporary Reservations

If a purchase is canceled without confirmation, instead of confirmation, call `/v1/Reservation/ExtReleaseReservations` with the parameter:

```json
{
  "externalReservationsIds": ["e56a9c34-6ab3-4817-b013-d2d31eecd457"]
}
```

This will return only _true/false_. It releases the reserved seat for other customers, and there is no need to wait for the expiration.
