Partition = Network Failure.

|   |   |
|---|---|
|**Partition**|The network connection between servers breaks|
|**Partition Tolerance**|The system keeps working anyway|
|**Trade-off**|If you keep working, you might have inconsistent data|
|**Reality**|In distributed systems, you HAVE to be partition tolerant (networks WILL fail)|

So when you have inconsistent data, you have two choices when a request comes to you.
Either return inconsistent data - You're having availability
return error and make them wait until partition recovers - You're having consistency


**Elevator System**
Elevator,Floor,Button,Door(Open/Closed),Request
**ParkingLot**
Spot,SpotType,Floor,Ticket,ParkingController,Vehicle,VehicleType,Gate
**Food Delivery**
User,DeliveryPartner,FoodItem,Restaurant,Order,OrderBookingController,Transaction,Reciept,Menu,Cart,Location