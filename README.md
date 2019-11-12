
# Як створювати інтерактивні карти з власними векторними тайлами (vector tile) за допомогою mapbox-gl.js 

Для чого це потрібно? Векторні тайли дозволяють створювати карти з __величезною кількістю елементів__ і запрограмувати їм __бажані стилі__. Карти при цьому не втрачають в продуктивності і легко будуть завантажуватися як на десктопах так і на мобільних телефонах без суттєвих затримок.  

Наприклад, ми використали векторні тайли для карти результатів другого туру Президентських виборів в Україні весною 2019 року.

[![Карта виборів Президента](image.png)](http://texty.org.ua/d/2019/president_elections_v2/)


## Частина перша. Як створити кастомний векторний тайлами (vector tile) для карти.

### Важливо
*Якщо ви не хочете встановлювати собі додатково програми, що потрібні для того, щоб створювати векторні тайли. Ви можете використати [docker image](https://hub.docker.com/repository/docker/ptrbdr/vector_tiles_image), який ми створили для роботи з тайлами. В ньому вже встановлено всі необхідні залежності для того, щоб перетворити geojson на векторні тайли.*

[Vector tile](https://docs.mapbox.com/vector-tiles/reference/ "vector tiles mapbox") або векторний тайл, це ефективний спосіб зберігати і використовувати геодані. Ми будемо користуватися векторними тайлами у форматі mbtiles. Це архів всередині якого знаходяться бінарні файли. Кожен файл описує всі елементи, що видно на карті при певному зумі в певній точці. Це така "плитка" на яку розбирата вся карта. На зумі 1, весь свій це одна "плитка", чим більший зум тим більше потрібно "плиток", щоб покрити всю карту і, відповідно тим більшими і важчим буде mbtiles архів, але з іншого боку можна буде відобразити більше деталей на карті.  

Детальніше про те якими бувають зуми (великими і маленькими) можна почати за на сайті бібліотеки 
[leaflet.js](https://leafletjs.com/examples/zoom-levels/ "leaflet docs")  

Більшость інструментів, які ми будемо використовувати створила команда mapbox, на їх сайті можна знайти документацію, описи і поради. Перше, що нам потрібно робити для того, щоб почати працювати з векторними тайлами це перетворити наш файл з геопросторовими даними в тайл. Для прикладу, ви можете скористатися файлом example.geojson, з цього репозиторія.  

Я намалював його на цьому [сайті](http://geojson.io/ "geojson.com"). В цьому файлі кілька полігонів, ліній і маркерів в Києві і околицях. Що ми з ним зробимо:  

* Перетворимо файл з геоданими в проекцію EPSG:4326.
* Перетворимо geojson або шейпфайл (.shp) з правильною проекцією його в mbtiles.
* Перевірити чи в mbtiles є ті елементи, що нам потрібні і, що вони відображаються правильно.
* Розархівуємо mbtiles.


Для того, щоб змінити проекцію вам можна використати утиліту [ogr2ogr](https://gdal.org/programs/ogr2ogr.html "ogr2ogr"). Вона є частиною бібліотеки [GDLA](https://gdal.org/ "GDAL"). За посилання на утиліту можна почитати її документацію і подивитися, що означають різні параметри. У нашому випадку, потрібна одна проста команда: 

  `ogr2ogr -f GeoJSON example_4326.geojson -t_srs EPSG:4326 example.geojson`  

__Параметри:__  

  `example.geojson` - Файл який ми хочемо перетворити в потрібний формат/проекцію
      
  `example_4326.geojson` - Назва файлу якого ми хочемо отримати на виході. Важливо, що ogr2ogr не стане перезаписувати файл, якщо у папці вже буде файл з такою ж назвою і викине вам помилку.  
  
  `f` - Назва формату в який ви хочете отримати на виході.  
  
  `t_srs` - Проекція в яку ви хочете перевести свій файл.  


Далі нам потрібно перетворити файл у формат mbiles, тобто з geojson зробити з нього векторний тайл. Це можна зробити за допомогою утиліти [tippecanoe](https://github.com/mapbox/tippecanoe "tippecanoe") яку створити розробники mapbox. Перш ніж працювати потрібно скопіювати репозиторій з утилітою `git clone https://github.com/mapbox/tippecanoe.git`, зайти в папку репозиторію `cd tippecanoe`, та запустити команди `make -j` та `make install`. 



Нижче опишемо як працювати з tippecanoe:  

  `tippecanoe -o example_tile.mbtiles -z12 -Z4 --coalesce-densest-as-needed --generate-ids example_4326.geojson`  

__Параметри:__  

  `example_4326.geojson`  - Файл з просторовими даним, що ми хочемо перетворити в тайл.
  
  `example_tile.mbtiles`  - Тайл, який ми отримуємо на виході.
  
  `-o`  - Параметр, що позначає назву вихідного тайла.
  
  `-z12`  - Максимальний рівень зуму для якого генеруються тайли. Чим він більший тим важчими стають тайли. Дефолтне значення 14. Але якщо прибрати 13 і 14 зум, можна суттєво зекономити на розмірах файлу.
  
  `-Z4`  - Мінімальний зум для якого генеруються тайли.
  
  `--coalesce-densest-as-needed`  - динамічно змінює розміри полігонів на різних зумах. Завдяки цьому тайли будуть не більшими ніж 500кб (ліміт браузера).
  
  `--generate-ids` - додає до кожного елемента унікальний ID. Потрібне для того, щоб задавати їм стилі.  
  
  
Після того як ви створите тайли, варто перевірити чи вони правильно відображаються і чи там є всі потрібні елементи. Це можна зробити за допомогою тайл сервера в докерівському контейнері. Якщо у вас немає докера, або ви не хочете витрачати час, можна повністю пропустити цей крок.  

Якщо ви все ж вирішили спробувати. Спершу потрібно завантажити контейнер з тайл-сервером.  


`docker pull klokantech/tileserver-gl`  

А потім запустити його в папці з проектом.

`docker run -it -v $(pwd):/data -p 8080:80 klokantech/tileserver-gl`  

Після цього можна відкрити у браузері сторінку `http://localhost:8080` і перевірити свої тайли.  

Все, що ми робили вище я дізнався з цього [туторіалу](https://openmaptiles.org/docs/generate/custom-vector-from-shapefile-geojson/ "openmaptiles")

Якщо ваш проект це дозволяє (там буде тайл-сервер), то цих кроків достатньо, щоб почати писати стилі для своєї карти. Але зазвичай потрібно уникнути сервера. Для цього потрібно розархівувати mbite та захостити папку, що утворилася на Github Pages. 

[mb-util](https://github.com/mapbox/mbutil "mb-util") дозволяє перетворити mbite файл в статичну папку з тайлами. Для цього вам потрібно просто клонтувати репозиторій і запустити mb-util. 

Код який потрібно запустити. Спочатку в папці де mbile файл, а тоді перейти в середину створеної папки (тут example) і запустити дві наступні. Вони розпаковують і перейменовують. 

`./mb-util --image_format=pbf countries.mbtiles countries`  

`gzip -d -r -S .pbf *`  

`find . -type f -exec mv '{}' '{}'.pbf \;`  

`mv metadata.json.pbf metadata.json`  

Тепер папка example — статичний векторний тайл, який можна викласти на Github Pages і використовувати за посилання, про що далі.

## Частина друга. Як застайлити векторний тайл.


Після того як ви створили тайли вам потрібно підключити їх до своїх сторінки. Для цього ми використаємо бібліотеку mapbox-gl.js.

Для того, щоб сторити карту підкладку достатньо додати ось цей шматочок коду. Він працює за схожим принципом як і у leaflet.js Ми задаємо можливий зум, центр та загальні стилі для підкладки та усіх шарів. 
```javascript
var map = new mapboxgl.Map({
    container: 'map',
    style: 'style3.json',
    minZoom: 4, //restrict map zoom
    maxZoom: 12,
    zoom: 6,
    center: [32.259271, 48.518688], 
    hash: false,
    tap: false
});
```


Далі, на подію "load" ми додаємо наші джерела даних та шари зі специфічними стилями. Важливо, що ми можемо мати одне джерело тайлів (як у випадку з картою виборів Президента 2019 року), але зробити з нього кілька шарів з окремими стилями і поведінкою.

Наприклад нижче шар для меж виборчих дільниць. Тут ми уточнюємо, що ширина лінії "line-width" має змінюватися залежно від зуму. На найменшому зумі, на якому видно всю Україна — 6, ми взагалі ні не додаємо цих ліній. На більших зумах ми додаємо ширину 0.4.


```javascript
	map.addLayer({
		"id": "election_districts_lines",
		"type": "line",
		"source": "election_districts",
		"source-layer": "simplified_4326",

		"paint": {
			"line-width": [
				"interpolate", ["linear"], ["zoom"], 
				6, 0,
				8, 0.4 
			],
			"line-color": "#ddd"
		}
	});
  
```

Загалом стилі задаються як `json` файли. Їх можна подавати прямо в коді сторінки або виносити в окремий файли. Загалом вони нагадують `css`, але є і відмінності.У mapbox є сторінка де описано специфікації для таких [стилів](https://docs.mapbox.com/mapbox-gl-js/style-spec/ "style-spec"). Окремо варто звернути увагу на [expressions](https://docs.mapbox.com/mapbox-gl-js/style-spec/#expressions "expressions") — це синтаксис для створення інтерактивних параметрів стилів за допомогою основних логічних виразів і порівняння різних даних між собою. Також ми використовуватимемо `expressions` для того, щоб фільтрувати дані вже в самому додатку.

Для фільтрування даних `js` можна використати такий синтаксис:


``` javascript
variable = 60;

map.setFilter('election_districts', ['>', ['get', 'zelenski'],
    variable
);

```
Він відфільтрує всі шари в яких у полі `zelenski` менше 60%.  


