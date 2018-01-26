### Quicker way to do it?

Sometimes it happens that when you try to run composer on VPS's it kills the process.
This is because not having enough RAM for the process. You can fix this using swap.
Here is how to achieve that very easily.

```zsh
free -m
mkdir -p /var/_swap_
cd /var/_swap_
dd if=/dev/zero of=swapfile bs=1M count=2000
mkswap swapfile
swapon swapfile
chmod 600 swapfile
echo "/var/_swap_/swapfile none swap sw 0 0" >> /etc/fstab
free -m
```
### The right way

- In live server don't ever run ```composer update```
- Run composer update on the dev/staging/local environment 
or somewhere else having same os and configurations. 
- upload the composer.lock file driectly, or using version control(git), to production(live server)
- run ```composer install``` on the  production(live server)

Want your own vps? use [MY REFERRAL LINK](https://m.do.co/c/734189696bdd) and get $10 in credit on registration.

