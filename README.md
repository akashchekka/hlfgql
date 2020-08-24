Hello there mates. Hope everyone's safe out there. This is Akash, working as Systems Engineer in TCS where I do Security Analysis for applications. Keeping that aside, I am a blockchain developer working on Hyperledger Fabric.

Chaincode is the smart contract that we write to implement business logic. Inorder to make the users to interact with chaincode we have cli tool and to access from various other applications we have many SDKs written in many languages. When we have end-to-end application, one way we interact with fabric network is using APIs. Here in this article I would like to show a new way of interacting with fabric network using GraphQL and how we can make Ultra-Rich queries using the same. I will give some basic introduction on what graphql is. I assume people reading this article has worked with hyperledger fabric atleast as a beginner.

### Scenario
Say for example we have a schema for user: Username, Email, Contact, Gender. If we need only Username and Gender, we need to write an API for returning those two. If we need email and contact, we need to write an API for that. And if we want all together there will be another API for that. But with GraphQL we can get whatever we need by just exposing one API. Lets see how!!

### What is GraphQL?
GraphQL is an open-source data query and manipulation language for APIs and a runtime for fulfilling queries with existing data. With GraphQL we can specify what exactly we need as part of query. GraphQL solves both over-fetching and under-fetching issues by allowing the client to request only the needed data. Since the client now has more freedom in the fetched data, development is much faster with GraphQL.

### SETUP

For this article I made use of fabcar that is part of fabric-samples provided by hyperledger fabric. In this fabcar chaincode, we have a schema for CAR with following fields: Make, Model, Colour, Model. Also we have invoke.js and query.js which uses fabric-network npm module to interact with peers and orderers. I will be making some modifications to query and invoke files to make it useful as required. Let's clear the network and remove stopped containers, prune the docker volume and network so that there wont be any hurdles in the process. To perform this, execute below script in fabcar directory.

```sh
$ ./networkDown.sh
```

Next lets start the network

```sh
$ ./startFabric.sh
```

Now you have the network up and running with peers, orderers, CAs and couchDBs. Next we shall make some minor modifications to the query and invoke scripts. Create a new directory named gql in fabcar directory and initiate a new npm package.

```sh
$ mkdir gql
$ cd gql
$ npm init
```
Give all the necessary details. Install all the necessary modules. Here we are writing an express application.

```sh
$ npm i express express-graphql graphql-tools fabric-network
```

Next we shall write basic express app.

```js
//index.js

import express from 'express';

const PORT = process.env.PORT || 3000

const app = express();
app.use(express.urlencoded({ extended: true }));
app.use(express.json());

app.use((err, req, res, next) => {
    res.locals.error = err;
    if (err.status >= 100 && err.status < 600) {
        res.status(err.status);
    } else {
        res.status(500);
        res.json({
          error: err
        })
    }
});

app.listen(PORT, () => console.log(`Running server on port localhost:${PORT}`));


```

We have two things with which we can interact with GraphQL: Query and Mutation. Query is equivalent to GET method in REST and Mutation is equivalent to POST,PUT. We have 'typedefs' in graphql where we specify the schema for data and 'resolvers' where we write logic part as shown below. We import two modules namely express-graphql and graphql-tools.

```js
// add this to query.js
import { graphqlHTTP } from 'express-graphql';
import { makeExecutableSchema } from 'graphql-tools';

const fabcar_typeDefs = `
    type Car {
        make: String
        model: String
        colour: String
        owner: String
    }

    type Query {
        QueryCar(name: String): Car
        QueryAllCars: [Car]
    }
`

const fabcar_resolvers = {
    Query: {
        QueryCar: (_, { name }) => {
            return new Promise(async (resolve, reject) => {
                var result = await query(['queryCar', name])
                console.log(JSON.parse(result))
                resolve(JSON.parse(result))
            })
        },
        QueryAllCars: () => {
            return new Promise(async (resolve, reject) => {
                var result = await query(['queryAllCars'])
                var parsedResult = JSON.parse(result)
                var response = []
                parsedResult.forEach(data => {
                    response.push(data.Record)
                })
                console.log(response)
                resolve(response)
            })
        }
    }
}

const schema = makeExecutableSchema({
    typeDefs: [
        fabcar_typeDefs
    ],
    resolvers: [
        fabcar_resolvers
    ]
});
```

Here we have a typeDef type 'Car' with fields: make, model, colour, model. Also we specified type Query which implements two methods QueryCar which takes 'name' input and returns Car, QueryAllCars which returns array of Cars. Resolvers implemented the logic

### DATA EXTRACTION

lets take blockfile_000000 from orderer. It can be seen that fabcar chaincode is using json.Marshal() function to convert data to bytes before writing data to ledger. I wrote a small code in js to extract json from a given string. Copying block file to our desired location and executing the js code will give us the json data that is present in ledger as you can see.
Here is the small piece of code that helps us extracting json from a given string.


```js
const fs = require('fs')
const extract = require('extract-json-from-string')
console.log(extract(fs.readFileSync(process.argv[2], "ascii")))
```
Here we are passing the file as a command line argument. Passing blockfile as input results following output.

```sh
$ node extract.js ./blockfile_000000
[ { make: 'Toyota',
    model: 'Prius',
    colour: 'blue',
    owner: 'Tomoko' },
  { make: 'Ford', model: 'Mustang', colour: 'red', owner: 'Brad' },
  { make: 'Hyundai',
    model: 'Tucson',
    colour: 'green',
    owner: 'Jin Soo' },
  { make: 'Volkswagen',
    model: 'Passat',
    colour: 'yellow',
    owner: 'Max' },
  { make: 'Tesla', model: 'S', colour: 'black', owner: 'Adriana' },
  { make: 'Peugeot',
    model: '205',
    colour: 'purple',
    owner: 'Michel' },
  { make: 'Chery', model: 'S22L', colour: 'white', owner: 'Aarav' },
  { make: 'Fiat', model: 'Punto', colour: 'violet', owner: 'Pari' },
  { make: 'Tata',
    model: 'Nano',
    colour: 'indigo',
    owner: 'Valeria' },
  { make: 'Holden',
    model: 'Barina',
    colour: 'brown',
    owner: 'Shotaro' } ]
```

Here we can see that the data we extracted is actually the one which we stored during initialising ledger. Eventhough I didn't enroll Admin or a user to invoke QueryAllCars function, I can visualize the data that is stored in ledger. If a hacker can compromise the server in which either a peer or an orderer is running, he can easily run his choice if he has a basic knowledge on fabric architecture. This may be very dangerous if it is a sensitive application.

### WHAT CAN WE DO?

I think you all have a clear idea about encryption and its advantages. Encryption can be one of many solutions to avoid unforeseen situations as mentioned in my article above. One can have a glance at [this](https://github.com/yeasy/docker-compose-files/blob/master/hyperledger_fabric/v2.1.0/examples/chaincode/go/enccc_example/utils.go) to get an idea on how to utilize encryption algorithms while implementing a chaincode. Ciphertext is hard to understand, making it impossible for one to extract cipher data. 

### CONCLUSION

This article conveys that data can be misused when certain measures are not taken. Many developers concentrate mostly on network and application development part but do not pay much attention when it comes to security criteria. This may not happen in the exact same way as mentioned but it doesn't cost you to be a little careful. In my coming blogs ill dig a little deeper about few more security insights.
