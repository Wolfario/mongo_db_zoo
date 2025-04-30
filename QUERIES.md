# 15 Otázek:
>Před zadáním otázek musíte spustit všechny příkazy ze souboru `HOWTO.md`

1. Najděte mezi našimi údaji **3 nejčastějších druhy zvířat**. Zapište druh zvířete a množství.

```js
db.animals.aggregate([
  // Skupina dokumentů podle druhu zvířat a spočítání počtu pro každý druh
  { $group: { _id: "$species", count: { $sum: 1 } } },
  
  // Seřazení výsledků sestupně podle počtu zvířat
  { $sort: { count: -1 } },
  
  // Omezení výsledků na prvních 3 záznamy
  { $limit: 3 }
])
```

2. **Zjistěte počet** všech `Cat, ringtail`, které jsou **středně nebo těžce nemocné** (`minor` nebo `sick`) a také je vypište.

```js
// Vypis takových koček
db.animals.find( { "species" : "Cat, ringtail", $or: [{ "health_status" : "minor" }, { "health_status" : "sick" }] } ).pretty()

// Jejích počet
db.animals.countDocuments( { "species" : "Cat, ringtail", $or: [{ "health_status" : "minor" }, { "health_status" : "sick" }] } )
```

3. **Přidejte nový atribut** `age` pro všechna zvířata v naší databázi, což bude znamenat, **jak staré je zvíře** k *01.01.2023* a **vypište nejstarší zvíře**.

```js
// Aktualizace mnoha dokumentů v kolekci "animals" pomoci updateMany
db.animals.updateMany(
   {},
   [
      {
         // Přidání nového pole "age" do každého dokumentu
         $addFields: {
            "age": {
               // Výpočet věku na základě odhadovaného roku narození
               $subtract: [
                  {$year: new Date("2023-01-01")},
                  {$year: "$approx_birth"}
               ]
            }
         }
      }
   ]
)

// Vypis nejstaršího zvířeti
db.animals.find({}).sort({ "age" : -1 }).limit(1)
```

4. **Změňte typ atributu ceny** z `string` na `double`. Také typ atributu **oblíbeného jídla** ze `string` na `array`.

```js
db.animals.updateMany(
   {},
   [
      {
         // Nastavení nového pole "price" s převedením řetězce na číslo
         $set: {
            "price": {
               $toDouble: {
                  $substr: [ "$price", 1, -1 ]
               }
            }
         }
      }
   ]
)

db.animals.updateMany(
   {},
   [
      {
         // Nastavení pole "favorite_dish" s hodnotou z původního pole
         $set: {
            "favorite_dish": ["$favorite_dish"]
         }
      }
   ]
)
```

5. **Uzdravte všechny** `Cat, ringtail` (`health_status`: `healthy`), kteří jsou **středně nebo těžce nemocní** (`health_status`: `minor` nebo `sick`). Poté **přidejte maso všem zdravým kočkám do jejich oblíbených pokrmů**, protože kočky by přece měly maso milovat (push `meat` do `favorite_dish`).

```js
db.animals.updateMany(
   { 
      // Pokud je druh zvířete "Cat, ringtail" a zdravotní stav je buď "minor" nebo "sick"
      "species": "Cat, ringtail", 
      "health_status": { $in: ["minor", "sick"] } 
   },
   {
      // Nastavení nového zdravotního stavu na "healthy"
      $set: {
         "health_status": "healthy"
      }
   }
)

db.animals.updateMany(
   // Pokud je druh zvířete "Cat, ringtail", zdravotní stav je "healthy" a "favorite_dish" již neobsahuje "meat".
   { 
      "species": "Cat, ringtail", 
      "health_status": "healthy",
      "favorite_dish": { $nin: ["meat"] }
   },
   {
      // Přidání oblíbeného jídla "meat" do pole "favorite_dish" :D
      $push: {
         "favorite_dish": "meat"
      }
   }
)
```

6. U všech zvířat, jejichž pohlaví je `null`, **nastavte pohlaví** na `N` a pro zvířata, která nemají `favorite_dish` (mají v `array` pouze hodnotu `null`), nastavte na `meat`

```js
/// Update null na N (neutrální) pohlaví
db.animals.updateMany(
   { gender: null },
   { $set: { gender: "N" } }
)

// Přídaní meat do favorite_dish pouze s null hodnotou
db.animals.updateMany(
  { favorite_dish: { $size: 1, $elemMatch: { $eq: null } } },
  { $set: { "favorite_dish.$[elem]": "meat" } },
  { arrayFilters: [{ "elem": null }] }
)
```

7. Vytvořte validační schéma pro `animals`, kde nakonfigurujete každý atribut tak, aby měl svůj typ a také `price` **nesmí být menší než 0** a pole `favorite_dish` **nemůže být prázdné**. Zkuste přidat neplatný dokument.

```js
// Nastavení validátoru schématu pro kolekci "animals"
db.runCommand({
   collMod: "animals",
   validator: {
      $jsonSchema: {
         bsonType: "object",
         required: ["id", "species", "name", "color", "gender", "zoo_address", "approx_birth", "health_status", "favorite_dish", "price"],
         properties: {
            id: {
               bsonType: "int"
            },
            species: {
               bsonType: "string"
            },
            name: {
               bsonType: "string"
            },
            color: {
               bsonType: "string"
            },
            gender: {
               bsonType: "string",
               "enum": ["M", "F", "N"]
            },
            zoo_address: {
               bsonType: "string"
            },
            approx_birth: {
               bsonType: "date"
            },
            health_status: {
               bsonType: "string"
            },
            // Požadované pole "favorite_dish" s datovým typem "array" a minimálním počtem prvků 1
            favorite_dish: {
               bsonType: "array",
               minItems: 1
            },
            // Požadované pole "price" s datovým typem "double" a minimální hodnotou 0
            price: {
               bsonType: "double",
               minimum: 0
            }
         }
      }
   },
   // Akce, která se provede při nesplnění validace - v tomto případě způsobí chybu
   validationAction: "error"
})

// Přidání neplatného dokumentu
db.animals.insertOne({
   "id": 123456,
   "species": "Cat",
   "name": "Whiskers",
   "color": "Gray",
   "gender": "M",
   "zoo_address": "123 Main Street",
   "approx_birth": new Date("2019-05-15"),
   "health_status": "Healthy",
   "favorite_dish": [], // Empty array
   "price": -5 // Negative value of price
})
```

8. **Vytvořte kolekci** `dishes`, která bude obsahovat potravu ze stravy pro naše zvířata. Vytvořte validační schéma tak, že
**každý dokument musí mít atributy** `name` (název potraviny), `health_influence` (jedna z hodnot: `heavy food`, `medicinal product`, `light food`, `healthy food`),
a také atribut `price` - náklady na jídlo.

```js
// Vytvoření kolekce dishes s definovaným validátorem schématu
db.createCollection("dishes", {
   validator: {
      $jsonSchema: {
         bsonType: "object",
         required: ["name", "health_influence", "price"],
         properties: {
            name: {
               bsonType: "string"
            },
            // Pole "health_influence" s datovým typem "string" a omezením hodnot pomocí enum
            health_influence: {
               bsonType: "string",
               enum: ["heavy food", "medicinal product", "light food", "healthy food"]
            },
            price: {
               bsonType: "double"
            }
         }
      }
   },
   validationAction: "error"
})

// Vložení více dokumentů do kolekce "dishes" s různými jídly, vlivem na zdraví a cenou
db.dishes.insertMany([
   { "name": "grass", "health_influence": "light food", "price": 1.25 },
   { "name": "leaves", "health_influence": "light food", "price": 1.75 },
   { "name": "seeds", "health_influence": "light food", "price": 2.55 },
   { "name": "fruits", "health_influence": "healthy food", "price": 5.15 },
   { "name": "nectar", "health_influence": "healthy food", "price": 3.40 },
   { "name": "insects", "health_influence": "heavy food", "price": 10.15 },
   { "name": "worms", "health_influence": "light food", "price": 8.65 },
   { "name": "meat", "health_influence": "heavy food", "price": 15.55 },
   { "name": "blood", "health_influence": "light food", "price": 4.45 },
   { "name": "carrion", "health_influence": "heavy food", "price": 16.20 },
   { "name": "pills", "health_influence": "medicinal product", "price": 12.50 }
])
```

9. Ke každému zvířeti v kolekci `animals` **přidejte atribut** `daily_ration`, což bude pole. Pro každé zvíře přidejte k jeho `daily_ration` libovolné jídlo (`name` z `dishes`), které má `price` menší než *12.50*.

```js
// Očekává se, že každý dokument v kolekci obsahuje pole "daily_ration"
// Pole "daily_ration" musí být typu pole (array) a je povinné
db.runCommand({
   collMod: "animals",
   validator: { $jsonSchema: {
      bsonType: "object",
      required: ["daily_ration"],
      properties: {
         daily_ration: {
            bsonType: ["array"],
         }
      }
   } }
})

// Jmena všech dishes která mají price menší než 12.5
let dish_names = db.dishes.find(
    { price: { $lt: 12.5 } },
    { name: 1, _id: 0 }
).toArray().map(dish => dish.name);

// Pro každé animal z kolekce animals
db.animals.find({}).forEach(function(animal) {
   // Nalezení nahodně vybraného jídla v kolekci dishes s cenou menší než 12.5
   let random_dish = dish_names[Math.floor(Math.random() * dish_names.length)];

   // Nastavení pole "daily_ration" na obsah jmena vybranéhoch jídla
   db.animals.updateOne({ _id: animal._id }, { $set: { daily_ration: [random_dish] } });
});
```

10. Přidejte do `daily_ration` pro všechna zvířata jejich oblíbená jídla (elementy z `favorite_dish`).

```js
// Přidání prvků z pole favorite_dish do pole daily_ration (addToSet ošetřuje opakování)
db.animals.find().forEach(function(animal) { // Loop přes každé zvíře
   animal.favorite_dish.forEach(function(dish) { // Loop přes každé jídlo
      db.animals.updateOne(
         { _id: animal._id },
         { $addToSet: { daily_ration: dish } }
      );
   });
});
```

11. **Vypište jméno, druh a také celkové náklady** na `daily_ration` zvířete, u kterého budou tyto náklady největší.

```js
db.animals.aggregate([
    {
        // Rozbalení pole "daily_ration" na jednotlivé hodnoty
        $unwind: "$daily_ration"
    },
    {
        // Propojení dokumentů z kolekce "animals" s dokumenty z kolekce "dishes" na základě názvu jídla
        $lookup:
        {
            from: "dishes",
            localField: "daily_ration",
            foreignField: "name",
            as: "dish_info"
        }
    },
    {
        // Rozbalení pole "dish_info" pro další manipulaci s informacemi o jídle
        $unwind: "$dish_info"
    },
    {
        // Seskupení podle ID zvířete, součet cen jídel a uchování informací o zvířeti
        $group:
        {
            _id: "$_id",
            total_price: { $sum: "$dish_info.price" },
            animal_info: { $first: "$$ROOT" }
        }
    },
    {
        // Seřazení výsledků sestupně podle celkové ceny denní stravy
        $sort: { total_price: -1 }
    },
    {
        // Omezení výsledků na první záznam
        $limit: 1
    },
    {
        // Projekce výsledků pro získání požadovaných polí
        $project:
        {
            _id: 0,
            name: "$animal_info.name",
            species: "$animal_info.species",
            total_price: 1
        }
    }
])
```
12. U všech zvířat, jejichž zdravotní stav je `sick` nebo `minor`, odstraníme z jejich daily_ration takové pokrmy, které
mají `health_influence`: `heavy food` a poté pro taková zvířata, která mají `health_status`: `sick`, přidejte do jejich `daily_ration` takové jídlo, které má
`health_influence`: `medicinal product`

```js
// Vytvoření pole jídel označených jako heavy food
let heavy_food = db.dishes.find({ health_influence: "heavy food" }).toArray().map(dish => dish.name);

db.animals.updateMany(
    // "health_status" rovno sick nebo minor
    { health_status: { $in: ["sick", "minor"] } },

    // Odebrání jídel označených jako heavy food z pole daily_ration
    { $pull: { daily_ration: { $in: heavy_food } } }
);

// Nalezení léčivého jídla označeného jako medicinal product'
let medicinal_dish = db.dishes.findOne({ health_influence: 'medicinal product' });

// Přidání názvu léčivého jídla do pole daily_ration pro zvířata se stavem sick
db.animals.updateMany(
   { health_status: 'sick' },
   { $push: { daily_ration: medicinal_dish.name } }
);
```
13. Najděte takový `species` zvířete, mezí kterými je nejvíc takových které mají zdravotní stav: `sick` nebo `minor`. **Vypište 3 taková zvířata**, **jejich celkový počet** (nemocných kazdého `species`) a procent (s `sick` nebo `minor`) od vsech zvířat kazdého `species`. **Odstraňte všechna tato zvířata z naší db**, protože jsme je odvezli do nemocnice.

```js
db.animals.aggregate([
  { $match: { health_status: { $in: ["sick", "minor"] } } }, // Filtrování zvířat s sick nebo minor
  { $group: { _id: "$species", total: { $sum: 1 } } },
  { $sort: { total: -1 } },
  { $limit: 3 }
]).forEach(function(species) {
  var percent = (species.total / db.animals.countDocuments({ species: species._id })) * 100; // Vypočítáme procento nemocných mezi všemi zvířaty konkretního species
  print(species._id + ": " + species.total + " (" + percent.toFixed(2) + "% od všech)");
  db.animals.deleteMany({ species: species._id });
});

// Pokusíme se najít White-fronted capuchin, který byl top 1
db.animals.find({ species: 'White-fronted capuchin'})
```

14. Najděte 3 takové `species` které milují `worms` víc než kdokoli jiný (největší počet zvířat, která mají `worms` v pole `favorite_dish`). Přidejte `meat` všem zvířatům těch `species` v `daily_ration`, pokud je tam nemají.

```js
// Najděme takové species
const tmp_species = db.animals.aggregate([  
  { $match: { favorite_dish: "worms" } }, // Mají worms  
  { $group: { _id: "$species", count: { $sum: 1 } } }, // Jejích počet
  { $sort: { count: -1 } },
  { $limit: 3 }
]).toArray();

tmp_species.forEach(species => {
  // Zajitíme, že už maso nemají
  const tmp_animals = db.animals.find({ species: species._id, daily_ration: { $ne: "meat" } });

  tmp_animals.forEach(animal => {
    db.animals.updateOne({ _id: animal._id }, { $push: { daily_ration: "meat" } });
  });
});
```

15. Použijme jednoduchý příklad, kde odebereme *nectar* z `daily_ration` zvířete jménem *Cynthie*, poté simulujeme vypnutí uzlu a **zkontrolujeme, zda je změna uložena na jiném uzlu**.

```js
// Aktualizuje dokument v kolekci animals, kde je name rovno Cynthie,
// a odstraní nectar z pole daily_ration
db.animals.updateOne(
   { "name": "Cynthie" },
   { $pull: { "daily_ration": "nectar" } }
)

// Vypneme databázový server, vynuceně a bez čekání na dokončení aktuálních operací
db.adminCommand({ shutdown: 1, force: true })

// Zde se sami připojíme k novému Primary uzlu !!
exit (2x)

docker exec -it mongo2 bash

monhosh

use animals

db.auth("test", "pass")

// Najde dokument v kolekci animals, kde je name rovno Cynthie
db.animals.find({"name": "Cynthie"})

// Změny zůstali na svém místě
```

