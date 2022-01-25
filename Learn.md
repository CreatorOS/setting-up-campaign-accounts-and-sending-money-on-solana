# Setting up campaign accounts and sending money on Solana

In the previous 2 quests, we’ve looked at how to write a program on Solana, and how to deploy it to the solana blockchain.

In the process we’ve also installed some of the command line tools. Specifically, the solana-cli.

If you’ve not covered that already, you should complete those quests first.

We assume you’ve deployed the function that we wrote in the first quest. In this quest, we’ll create an account for a campaign and send money to the campaign account. In the next quest, we will integrate the accounts we create in this quest with the program we wrote in the first quest to add additional checks and balances.

To continue, make sure your Solana blockchain is running on one of the terminals using `solana-test-validator`.

Also make sure your program is deployed using `solana program deploy targets/deploy/crowdfunding.so`

For more details, refer to the previous quest in this series.

Now, on to building

## Getting the libraries ready

In this quest, we’ll be using javascript (node) to interact with solana and our program.

Open a directory in your terminal and initialize it as a node project using

```

npm init

```

We will be using 2 libraries. The first is `@solana/web3.js` which is the library where all the solana related methods lie. The second is a file management utility called `mz`.

```

npm install @solana/web3.js mz

```

Open up your favorite code editor to create a file `index.js`

First up, we’ll write a function to check if the blockchain is running correctly.

To do so, create a function called checkConnection.

```

async function establishConnection() {

  const rpcUrl = 'http://localhost:8899';

  connection = new Connection(rpcUrl, 'confirmed');

  const version = await connection.getVersion();

  console.log('Connection to cluster established:', rpcUrl, version);

}

```

`rpcUrl` is the address at which your solana blockchain is running. When you run `solana-test-validator`, it defaults to the port 8899. All the interactions with this blockchain must happen using RPC as a mode of communication.

In this function we use an object called `connection`. Import the appropriate module from `web3.js`

```

const { Connection } = require(‘@solana/web3.js’);

```

Finally at the end of the file call the function `establishConnection()`

Your file must now look like this : [https://gist.github.com/madhavanmalolan/a9bf52f73db791c65bb86c8e8c435e25](https://gist.github.com/madhavanmalolan/a9bf52f73db791c65bb86c8e8c435e25)

You can now run this file as `node index.js`

If all is well, you’ll see that the connection is established, with the version of Solana in the output of the above command.

## Who’ll pay?

Whenever you are sending transactions to Solana asking it to execute your instructions, like creating an account - you must pay some lamports for execution.

If the lamports (aka money) are in your account, you need to sign the transaction with your private key. That way no one else can spend your lamports.

This private key is stored in your machine. This is the file solana-cli has been using all this while to execute an airdrop, fetch account details etc (ref: Quest 2). You can see this at `~/.config/solana/id.json`

You can open it and you’ll see that it is an array of numbers. This is the encoded format of your private key. We will use this private key to sign transactions, and all the instructions we’ll execute will debit the account associated with this private key.

To fetch the private key from this file :

```

async function createKeypairFromFile() {

  const secretKeyString = await fs.readFile(“~/.config/solana/id.json”, {encoding: 'utf8'});

  const secretKey = Uint8Array.from(JSON.parse(secretKeyString));

  return Keypair.fromSecretKey(secretKey);

}

```

Most of the code in this function should be self explanatory. We’re reading the file that we just explored `id.json`. We’re converting it into a Keypair using `fromSecretKey`.

This keypair that is returned has a public key and a private key. The private key is always associated with a unique public key that can be calculated from the private key.

[https://gist.github.com/madhavanmalolan/39b62d66609b89e134bdf9abc8d53b75](https://gist.github.com/madhavanmalolan/39b62d66609b89e134bdf9abc8d53b75)

## Creating a campaign account public key (aka address)

Every account has a private key and a public key, just like what we saw in the previous quest.

Now we need to create an account specifically for a particular campaign. We’ll create multiple accounts, each pertaining to a unique campaign. That way, we’ll never mix up the money across two campaigns. Or, put another way, it will be impossible for the program to misuse the funds - thereby giving funders the trust that the money they’re sending can only be used for the intended purpose.

To do so, we will first create a new account that will be bound to the program we’ve deployed.

```

 const signer = await createKeypairFromFile();

 const newAccountPubkey = await PublicKey.createWithSeed(

    signer.pubkey,

    "campaign1",

    new PublicKey("J9iUgxytByjcsXwxMN47J3MDwGfNQFnDirxWnabTQWLY"),

  );

```

The first parameter is the account using which we’re going to create this account. We’ll create it using our default account.

The second parameter is what is called a seed phrase. If you remember from quest 2, when you created your account, you were given a seed phrase. This seed phrase can be used to construct the private key. So in essence, this seed phrase is a proxy to entering the private key of the new account.

The last parameter is the public key of the program you deployed. This is also called the program id. This account that we’re creating is going to be associated with this program. This program will control all the data and the lamports in this program. This is infact one of the more important concepts in Solana. These accounts we’ll create (by changing the seed phrase) are called PDAs - program derived accounts. All the accounts including your main account is controlled using a program. User accounts from which you can send and receive money is controlled by a special program called the SystemProgram.

## Creating the account on Solana

Now that we have generated a public key for our account, let’s tell Solana to make this account on the blockchain. Once this account exists on the blockchain, we’ll be able to interact with it using our program. Then we can apply our checks and balances to make sure that the funding is correctly received and disbursed.

To do so, we’ll first create the instruction. This is an instruction telling solana to create a new account.

```

  const instruction = SystemProgram.createAccountWithSeed({

    fromPubkey: signer.pubkey,

    basePubkey: signer.pubkey,

    seed: "campaign1",

    newAccountPubkey,

    lamports, // todo

    space: 1024,

    programId : new PublicKey("J9iUgxytByjcsXwxMN47J3MDwGfNQFnDirxWnabTQWLY"),

  });

```

Here, this is an instruction created that will be sent to the SystemProgram to create an account with the said public key `newAccountPubkey` and associate it with the program with the given programId. Now this program identified by the programId is the owner of this account.

We noted earlier that every account has the following

- Public key
- Private key
- Data
- Money

We’ve already have the public and private key.

While creating this account we need to allocate space to store the data. For this example, we’ve put in 1024 as the number of bytes we need to store any data in this function. It is arbitrary, but good enough to explain the concept at this stage. We will get scientific & exact about this in the next quest when we’re integrating with our program.

Now that we’ve allocated space to the account we’re creating, we also need to pay rent. On Solana it is not free to store information, you actually have to pay to keep the data and account alive. However, the rent is exempt if you store a minimum amount of lamports in this account. If you don’t pay the rent and you dont have the minimum balance, the account gets deleted after some time.

Howmuch money is required to be in the account to be exempted of rent is a function of how much space the account is requesting.

```

  const lamports = await connection.getMinimumBalanceForRentExemption(

    1024,

  );

```

We can use the function from the solana library to calculate the minimum amount.

This then feeds into our previous instruction construction.

## Sending the instruction

We’ve created the instruction to create an account, now we need to have the instruction executed on the blockchain.

Now, construct the transaction

```

  const transaction = new Transaction().add(

    instruction

  );

```

And lastly send the transaction.

```

  await sendAndConfirmTransaction(connection, transaction, [signer]);

```

The connection object is the same object that connects to the local instance of Solana.

The signer is who signs this transaction. If the transaction is not signed, it cannot be executed. By signing, you’re proving that the account creation request is indeed coming from you. The signature proves that the transaction is made by a user who has access to the private key of the public key mentioned in the new account creation instruction.

With that, your script is ready to be run again. `node index.js`

[https://gist.github.com/madhavanmalolan/6188cd29518b62826981e6f911737b80](https://gist.github.com/madhavanmalolan/6188cd29518b62826981e6f911737b80)

## Sanity Check & transferring money

You’ll see the public key of the account that has just been created displayed on your console.

As a quick sanity check, let’s see what is there in this account

```

solana account &lt;enter the above displayed address>

```

You’ll see something like this

```

Public Key: 8yejfSwzQbNBUroiuoepJKJEVitw1qwzhNH4VYWLGVUC                                                                                                                            Balance: 0.00801792 SOL                                                                                                                                                             Owner: J9iUgxytByjcsXwxMN47J3MDwGfNQFnDirxWnabTQWLY                                                                                                                                 Executable: false                                                                                                                                                                   Rent Epoch: 0                                                                                                                                                                       Length: 1024 (0x400) bytes                                                                                                                                                          0000:   00 00 00 00  00 00 00 00  00 00 00 00  00 00 00 00   ...............                  

```

The public key is the address of this account.

The balance is the number of lamports in this account. This is the lamports we transferred while creating the account to make sure it is rent exempt.

You’ll also notice that the owner of this account is the program. Only the program can make any changes to this account.

This account is not executable, because this is not a program account. However all the programs that you write, will reside in an account that will be executable.

Lastly the length of the data is 1024, just like what we had hardcoded.

Following are the 1024 bytes. All will be 0s because we’ve not set any data in this account yet.

Now you can start transferring money to this account.

```

solana transfer --from ~/.config/solana/id.json &lt;public key> 1 --fees-payer ~/.config/solana/id.json

```

This will transfer 1 sol from your account to the new account.

```

solana account &lt;enter public key of new account here>

```

If you run this command again, you’ll see that the account balance has actually increased by 1SOL.

Awesome!

So now you’ve created a new campaign account and have started sending money to it.

In the next quest, we’ll make sure the campaign data is correctly updated and that the correct person is able to withdraw the funds from this account.
