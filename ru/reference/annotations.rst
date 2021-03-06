Парсер аннотаций
==================
Это первый прецедент для мира PHP, когда компонент парсера аннотаций написан на C. Phalcon\\Annotations - компонент общего назначения, который обеспечивает простоту анализа и кэширования аннотаций для PHP классов, используемых в приложениях. 

Аннотации читаются из блоков документации в классах, его методах и свойствах. Аннотации могут быть помещены в любое место блока документации. 

.. code-block:: php

	<?php

	/**
	 * Это описание класса
	 *
	 * @AmazingClass(true)
	 */
	class Example
	{

		/**
		 * Это свойство с особенностью
		 *
		 * @SpecialFeature
		 */
		protected $someProperty;

		/**
		 * Это метод
		 *
		 * @SpecialFeature
		 */
		public function someMethod()
		{
			// ...
		}

	}

В примере выше мы показали аннотации в комментариях, имеющие следующий синтаксис:

@Annotation-Name[(param1, param2, ...)]

Аннотации также могут быть помещены в любую часть блока документации:

.. code-block:: php

	<?php

	/**
	 * Это свойство с особенностью
	 *
	 * @SpecialFeature
	 *
	 * Еще комментарии
	 *
	 * @AnotherSpecialFeature(true)
	 */

Парсер является очень гибким инструментом, поэтому следующий блок документации также является правильным:

.. code-block:: php

	<?php

	/**
	 * Это свойство с особенностью @SpecialFeature({
	someParameter="the value", false

	 })  Еще комментарии @AnotherSpecialFeature(true) @MoreAnnotations
	 **/

Тем не менее, рекомендуется помещать аннотации в конце блоков документации, чтобы сделать код более понятным и удобным в поддержке: 

.. code-block:: php

	<?php

	/**
	 * Это свойство с особенностью
	 * Еще комментарии
	 *
	 * @SpecialFeature({someParameter="the value", false})
	 * @AnotherSpecialFeature(true)
	 */

Чтение аннотаций
-------------------
Для простого получения аннотаций класса, используя объектно-ориентированный интерфейс, реализован рефлектор:

.. code-block:: php

	<?php

	$reader = new \Phalcon\Annotations\Adapter\Memory();

	//Отразить аннотации в классе Example
	$reflector = $reader->get('Example');

	//Прочесть аннотации блоке документации класса
	$annotations = $reflector->getClassAnnotations();

	//Произвести обход всех аннотаций
	foreach ($annotations as $annotation) {

		//Вывести название аннотации
		echo $annotation->getName(), PHP_EOL;

		//Вывести количество аргументов
		echo $annotation->numberArguments(), PHP_EOL;

		//Вывести аргументы
		print_r($annotation->getArguments());
	}

Процесс чтения аннотаций является очень быстрым. Тем не менее, по причинам производительности, мы рекомендуем хранить обработанные аннотации используя адаптер.
Адаптеры кэшируют обработанные аннотации, избегая необходимость в разборе аннотаций снова и снова.

:doc:`Phalcon\\Annotations\\Adapter\\Memory <../api/Phalcon_Annotations_Adapter_Memory>` был использован в примере выше. Этот адаптер
только кэширует аннотации в процессе работы, поэтому он более подходит для разработки. Существуют и другие адаптеры, 
которые можно использовать, когда приложение используется в продакшене. 

Типы аннотаций
--------------------
Аннотации могут иметь или не иметь параметров. Параметры могут быть простыми литералами (строки, числа, boolean, null), массивом, хешированным списком или другими аннотациями:

.. code-block:: php

	<?php

	/**
	 * Простая аннотация
	 *
	 * @SomeAnnotation
	 */

	/**
	 * Аннотация с параметрами
	 *
	 * @SomeAnnotation("hello", "world", 1, 2, 3, false, true)
	 */

	/**
	 * Аннотация с именованными параметрами
	 *
	 * @SomeAnnotation(first="hello", second="world", third=1)
	 * @SomeAnnotation(first: "hello", second: "world", third: 1)
	 */

	/**
	 * Передача массива
	 *
	 * @SomeAnnotation([1, 2, 3, 4])
	 * @SomeAnnotation({1, 2, 3, 4})
	 */

	/**
	 * Передача хеша в качестве параметра
	 *
	 * @SomeAnnotation({first=1, second=2, third=3})
	 * @SomeAnnotation({'first'=1, 'second'=2, 'third'=3})
	 * @SomeAnnotation({'first': 1, 'second': 2, 'third': 3})
	 * @SomeAnnotation(['first': 1, 'second': 2, 'third': 3])
	 */

	/**
	 * Вложенные массивы/хеши
	 *
	 * @SomeAnnotation({"name"="SomeName", "other"={
	 *		"foo1": "bar1", "foo2": "bar2", {1, 2, 3},
	 * }})
	 */

	/**
	 * Вложенные аннотации
	 *
	 * @SomeAnnotation(first=@AnotherAnnotation(1, 2, 3))
	 */

Практическое использование
---------------
Давайте представим что у нас есть контроллер и разработчик хочет сделать плагин, который автоматически запускает
кэширование если последнее запущенное действие было помечено как имеющее возможность кэширования. Прежде всего, мы зарегистрируем плагин в сервисе Dispatcher,
чтобы быть уведомленными, когда маршрут исполняется:

.. code-block:: php

	<?php

	$di['dispatcher'] = function() {

		$eventsManager = new \Phalcon\Events\Manager();

		//Привязать плагин к событию 'dispatch'
		$eventsManager->attach('dispatch', new CacheEnablerPlugin());

		$dispatcher = new \Phalcon\Mvc\Dispatcher();
		$dispatcher->setEventsManager($eventsManager);
		return $dispatcher;
	};

CacheEnablerPlugin это плагин, который перехватывает каждое запущенное действие в диспетчере, включая кэш если необходимо:

.. code-block:: php

	<?php

	/**
	 * Включение кэша для представления, если 
	 * последнее запущенное действие имело аннотацию @Cache
	 */
	class CacheEnablerPlugin extends \Phalcon\Mvc\User\Plugin
	{

		/**
		 * Это событие запускается перед запуском каждого маршрута в диспетчере
		 *
		 */
		public function beforeExecuteRoute($event, $dispatcher)
		{

			//Разбор аннотаций в текущем запущенном методе
			$annotations = $this->annotations->getMethod(
				$dispatcher->getActiveController(),
				$dispatcher->getActiveMethod()
			);

			//Проверить, имеет ли метод аннотацию 'Cache'
			if ($annotations->has('Cache')) {

				//Метод имеет аннотацию 'Cache'
				$annotation = $annotations->get('Cache');

				//Получить время жизни кэша
				$lifetime = $annotation->getNamedParameter('lifetime');

				$options = array('lifetime' => $lifetime);

				//Проверить, есть ли определенный пользователем ключ кэша
				if ($annotation->hasNamedParameter('key')) {
					$options['key'] = $annotation->getNamedParameter('key');
				}

				//Включить кэш для текущего метода
				$this->view->cache($options);
			}

		}

	}

Теперь мы можем использовать аннотации в контроллере:

.. code-block:: php

	<?php

	class NewsController extends \Phalcon\Mvc\Controller
	{

		public function indexAction()
		{

		}

		/**
		 * Это комментарий
		 *
		 * @Cache(lifetime=86400)
		 */
		public function showAllAction()
		{
			$this->view->article = Articles::find();
		}

		/**
		 * Это комментарий
		 *
		 * @Cache(key="my-key", lifetime=86400)
		 */
		public function showAction($slug)
		{
			$this->view->article = Articles::findFirstByTitle($slug);
		}

	}


