# mongodb

select count(*) from tabel; ~ db.tabel.count();

select * from table where phone= "84379352035"; ~ db.table.find({"phone":"84379352035"});

truncate table abc; ~ db.Crawl_Results.remove({});

update table; ~

db.Data.update({ _id: ObjectId("6326c9b2bd68bdfb71a6e560") }, {
    $set: {
        "name": "",
        "uid": "107",
        "gender": "",
        "birthday": "",
        "email": "",
        "phone": "84379352035",
        "location": "",
        "isCheck": false
    }
})


db.test.insert({x: 1})

db.test.find()                                               // no criteria
{ "_id" : ObjectId("4ecc05e55dd98a436ddcc47c"), "x" : 1 }      

db.test.find({"_id" : ObjectId("4ecc05e55dd98a436ddcc47c")}) // explicit
{ "_id" : ObjectId("4ecc05e55dd98a436ddcc47c"), "x" : 1 }

db.test.find(ObjectId("4ecc05e55dd98a436ddcc47c"))           // shortcut
{ "_id" : ObjectId("4ecc05e55dd98a436ddcc47c"), "x" : 1 }


Updating all documents:
syntax: db.collection.updateMany({}, {update})
db.Data.updateMany({}, {$set: { "isCheck": null }});
