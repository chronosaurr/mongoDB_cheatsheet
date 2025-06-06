# MongoDB Cheatsheet

_____________________________________________________


## 1. Podstawowe komendy

```javascript
show dbs                          // shows all databases
use my_database                   // switches to (or creates) a database
db.createCollection("clients")    // creates a new collection
show collections                  // shows collections in the active database
db.clients.drop()                 // deletes the "clients" collection
db.clients.countDocuments()       // counts documents in the collection
db.dropDatabase()                 // deletes the active database
 
```



## 2. Wstawianie dokumentów
```javascript
db.clients.insertOne({ firstName: "Jan", lastName: "Kowalski", age: 30 })

db.clients.insertMany([
  { firstName: "Anna", lastName: "Nowak", age: 25 },
  { firstName: "Piotr", lastName: "Zieliński", age: 40 }
])
```

## 3. Wyszukiwanie (find)
```javascript
db.clients.find()                                // find all documents
db.clients.find({ firstName: "Jan" })            // filter by field
db.clients.findOne({ age: { $gt: 30 } })         // one document where age > 30
db.clients.find().pretty()                       // formatted output
db.users.find().limit(5).sort({age: -1})         // sort by age, descending, limited to 5 results
```

### dodatkowo przydatne operatory
#### operatory porównania
- $eq – equal
- $ne – different
- $gt – greater than
- $gte – greater than or equal
- $lt – less than
- $lte – less than or equal
#### operatory logiczne
- $and
- $or
- $not
- $nor

na przykład
```javascript
db.customers.find({
  $and: [
    { registration_date: { $gt: "2021-12-31" } },
    { orders: { $ne: [] } },
    {
      $nor: [
        { "address.country": "France" },
        { "address.country": "Austria" }
      ]
    }
  ]
});
```
* `find()` domyślnie zwraca maksymalnie 20 dokumentów w shellu. Użyj `.toArray()` lub `.pretty()` dla pełnego wyniku.
* Użycie operatorów logicznych i porównawczych pozwala tworzyć bardzo precyzyjne zapytania.
* Kwerendy mogą być zagnieżdżane i łączone.

 
## 4. Aktualizacja dokumentów - update 
```javascript
db.users.updateOne(
  {name: "Jan"},
  {$set: {age: 31}}
)

db.users.updateMany(
  {age: {$lt: 30}},
  {$inc: {age: 1}}     // increase by 1
)

```
### operatory aktualizacji
- $set – setting new field values
- $inc – incrementing (or decrementing) numeric values
- $push – adding an element to an array
- $pull – removing elements from an array

w tym przykładzie podany dokument zastępuje cały dokument pod indeksem 7.
```javascript
db.customers.replaceOne(
  { source_id: 7 },
  {
    source_id: 7,
    first_name: "Natalia",
    last_name: "Szara",
    email: "natalia.szara@example.com",
    registration_date: "2025-05-28 10:00:00",
    address: {
      street: "Nowa 123",
      city: "Kraków",
      zip_code: "31-000",
      country: "Polska"
    },
    orders: []
  }
);
```
* `deleteOne()` przyjmuje ten sam filtr co `find()` – usuwa pierwszy pasujący dokument.
* `deleteMany()` może usunąć wiele dokumentów – warto stosować ostrożnie.



## 5. Usuwanie kolekcji
```javascript
db.users.deleteOne({name: "Jan"})
db.users.deleteMany({age: {$gt: 40}})
```
* deleteOne() przyjmuje ten sam filtr co find() – usuwa pierwszy pasujący dokument.
* deleteMany() może usunąć wiele dokumentów – warto stosować ostrożnie.

## 6. Wszystkich ukochana agregacja
```javascript
db.users.aggregate([
  {$match: {age: {$gte: 30}}},
  {$group: {_id: "$age", count: {$sum: 1}}}
])

```
### Przy aggregate wszystkie zadania można rozwiazać wykorzystując poniższe zapytania:
- $unwind,
- $group,
- $project,
- $match,
- $sort,
- $limit,
  
itd.

### ważne!!
* Etapy agregacji wykonują się w kolejności – każdy operuje na wynikach poprzedniego.
* Można łączyć wiele etapów w jednym pipeline.
* Wydajne agregacje często wykorzystują indeksy w pierwszym etapie `$match`.

Więcej zaawansowanych przykładów można uzyskać przez połączenie `$match`, `$group`, `$project` i `$unwind` w jednym zapytaniu.

2 problematyczne przykłady z zajęć wytłumaczone;
```javascript
// 3 Wykonawcy według gatunków
//Pogrupuj wykonawców według gatunku utworów i oblicz,
//ilu unikalnych wykonawców występuje w każdym gatunku.

db.artists.aggregate([
  { $unwind: "$albums" },                    //Rozdziela tablicę albumów: każdy album staje się osobnym dokumentem.
  { $unwind: "$albums.tracks" },             //Rozdziela tablicę utworów (tracks) z każdego albumu.
  {
    $group: {                                //Grupuje dokumenty według gatunku 
      _id: "$albums.tracks.genre",           //(_id: "$albums.tracks.genre"), dodając artystów do zbioru unique_artists.
      unique_artists: { $addToSet: "$Name" }
    }
  },
  {
    $project: {
      _id: 1,
      artist_count: { $size: "$unique_artists" }
    }                                        //Liczy liczbę unikalnych artystów dla każdego gatunku za pomocą $size
  }
])
```



```javascript
// 7. Średnia cena utworu per artysta
// Dla każdego wykonawcy oblicz średnią cenę
// (unit_price) wszystkich jego utworów.


db.tracks.aggregate([
  {
    $lookup: {
      from: "albums",                        // Dołącza dane z kolekcji albums
      localField: "albums_id",               // Porównaje wartości z pola albums_id
      foreignField: "id",                    // do pola id w kolekcji albums
      as: "albums"                           // Wynik dołącza jako tablicę albums
    }
  },
  { $unwind: "$albums" },                    // Rozdziela tablicę albums – każdy album staje się osobnym dokumentem
  {
    $lookup: {
      from: "artists",                       // Dołącza dane z kolekcji artists
      localField: "albums.artists_id",       // Łączy przez pole artists_id w albumie
      foreignField: "id",                    // i id w kolekcji artists
      as: "artists"                          // Wynik dołącz jako tablicę artists
    }
  },
  { $unwind: "$artists" },                   // Rozdziela tablicę artists – jeden dokument = jeden artysta, standardowo
  {
    $group: {
      _id: "$artists.name",                  // Grupuje po nazwie artysty
      average_price: { $avg: "$unit_price" } // Oblicza średnią cenę jego utworów
    }
  },
  {
    $sort: { average_price: -1 }             // Sortuje malejąco według średniej ceny
  }
])

```
