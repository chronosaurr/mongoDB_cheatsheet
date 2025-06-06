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
```$eq, $ne, $gt, $gte, $lt, $lte, $in, $nin```


 
## 4. Aktualizacja dokumentów
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

## 5. Usuwanie kolekcji
```javascript
db.users.deleteOne({name: "Jan"})
db.users.deleteMany({age: {$gt: 40}})
```

## 6. Wszystkich ukochana agregacja
```javascript
db.users.aggregate([
  {$match: {age: {$gte: 30}}},
  {$group: {_id: "$age", count: {$sum: 1}}}
])

```
### Przy aggregate wszystkie zadania można rozwiazać wykorzystując poniższe zapytania:
`$unwind, $group, $project, $match, $sort, $limit, itd.`


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
