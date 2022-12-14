**

for limit in fsize cpu as memlock

do

  grep "mongod" limits.conf | grep -q $limit || echo -e "mongod  hard  $limit  unlimited\nmongod  soft  $limit  unlimited" |  tee --append limits.conf

done

  

for limit in nofile noproc

do

  grep "mongod" limits.conf | grep -q $limit || echo -e "mongod  hard  $limit  64000\nmongod  soft  $limit  64000" |  tee --append limits.conf

done

**