---
title: 404
date: 2021-05-23 13:43:42
permalink: /404.html
---
<!-- markdownlint-disable MD039 MD033 -->

## 404 :(

Oops, page does not exist.

You will be directed to **[home page](https://arronhong.github.io/)** after <span id="timeout">5</span> seconds.

<script>
let countTime = parseInt(document.getElementById('timeout').textContent);

function count() {
  countTime -= 1;
  if(countTime === 0){
    location.href = 'https://arronhong.github.io/';
  }
  setTimeout(() => {
    count();
  }, 1000);
}

count();
</script>