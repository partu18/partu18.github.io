# Bashed 
## User

This machine was pretty easy.
First of all I used **dirb** to start listing some common directories.

```bash

dirb http://10.10.10.68 common.txt

```

Some directories were found:

![dirb directories](./images/bashed/dirb.png)

The **dev** seemed to be the most interesting one, therefore I started from there:

![dev index](./images/bashed/dev_index.png)

And once inside **phpbash.php**:

![dev index](./images/bashed/user_hash.png)

## Root


**TBD**