Потому что `i` никогда не станет равным `10`.

Запустите, чтобы увидеть *реальные* значения `i`:

```js
//+ run
var i = 0;
while(i < 11) { 
  i += 0.2;
  if (i>9.8 && i<10.2) alert(i);
}
```

Ни одно из них в точности не равно `10`. 