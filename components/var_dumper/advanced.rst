.. index::
   single: VarDumper
   single: Components; VarDumper

Advanced Usage of the VarDumper Component
=========================================

The ``dump()`` function is just a thin wrapper and a more convenient way to call
:method:`VarDumper::dump() <Symfony\\Component\\VarDumper\\VarDumper::dump>`.
You can change the behavior of this function by calling
:method:`VarDumper::setHandler($callable) <Symfony\\Component\\VarDumper\\VarDumper::setHandler>`.
Calls to ``dump()`` will then be forwarded to ``$callable``.

By adding a handler, you can customize the `Cloners`_, `Dumpers`_ and `Casters`_
as explained below. A simple implementation of a handler function might look
like this::

    use Symfony\Component\VarDumper\VarDumper;
    use Symfony\Component\VarDumper\Cloner\VarCloner;
    use Symfony\Component\VarDumper\Dumper\CliDumper;
    use Symfony\Component\VarDumper\Dumper\HtmlDumper;

    VarDumper::setHandler(function($var) {
        $cloner = new VarCloner();
        $dumper = 'cli' === PHP_SAPI ? new CliDumper() : new HtmlDumper();

        $dumper->dump($cloner->cloneVar($var));
    });

Cloners
-------

A cloner is used to create an intermediate representation of any PHP variable.
Its output is a :class:`Symfony\\Component\\VarDumper\\Cloner\\Data`
object that wraps this representation.

You can create a :class:`Symfony\\Component\\VarDumper\\Cloner\\Data`
object this way::

    use Symfony\Component\VarDumper\Cloner\VarCloner;

    $cloner = new VarCloner();
    $data = $cloner->cloneVar($myVar);
    // this is commonly then passed to the dumper
    // see the example at the top of this page
    // $dumper->dump($data);

A cloner also applies limits when creating the representation, so that the
corresponding Data object could represent only a subset of the cloned variable.
Before calling :method:`Symfony\\Component\\VarDumper\\Cloner\\VarCloner::cloneVar`,
you can configure these limits:

* :method:`Symfony\\Component\\VarDumper\\Cloner\\VarCloner::setMaxItems`
  configures the maximum number of items that will be cloned
  *past the first nesting level*. Items are counted using a breadth-first
  algorithm so that lower level items have higher priority than deeply nested
  items;
* :method:`Symfony\\Component\\VarDumper\\Cloner\\VarCloner::setMaxString`
  configures the maximum number of characters that will be cloned before
  cutting overlong strings;
* in both cases, specifying `-1` removes any limit.

Before dumping it, you can further limit the resulting
:class:`Symfony\\Component\\VarDumper\\Cloner\\Data` object by calling its
:method:`Symfony\\Component\\VarDumper\\Cloner\\Data::getLimitedClone`
method:

* the first ``$maxDepth`` argument allows limiting dumps in the depth dimension,
* the second ``$maxItemsPerDepth`` limits the number of items per depth level,
* and the last ``$useRefHandles`` defaults to ``true``, but allows removing
  internal objects' handles for sparser output,
* but unlike the previous limits on cloners that remove data on purpose,
  these can be changed back and forth before dumping since they do not affect
  the intermediate representation internally.

.. note::

    When no limit is applied, a :class:`Symfony\\Component\\VarDumper\\Cloner\\Data`
    object is as accurate as the native :phpfunction:`serialize` function,
    and thus could be for purposes beyond dumping for debugging.

Dumpers
-------

A dumper is responsible for outputting a string representation of a PHP variable,
using a :class:`Symfony\\Component\\VarDumper\\Cloner\\Data` object as input.
The destination and the formatting of this output vary with dumpers.

This component comes with an :class:`Symfony\\Component\\VarDumper\\Dumper\\HtmlDumper`
for HTML output and a :class:`Symfony\\Component\\VarDumper\\Dumper\\CliDumper`
for optionally colored command line output.

For example, if you want to dump some ``$variable``, just do::

    use Symfony\Component\VarDumper\Cloner\VarCloner;
    use Symfony\Component\VarDumper\Dumper\CliDumper;

    $cloner = new VarCloner();
    $dumper = new CliDumper();

    $dumper->dump($cloner->cloneVar($variable));

By using the first argument of the constructor, you can select the output
stream where the dump will be written. By default, the ``CliDumper`` writes
on ``php://stdout`` and the ``HtmlDumper`` on ``php://output``. But any PHP
stream (resource or URL) is acceptable.

Instead of a stream destination, you can also pass it a ``callable`` that
will be called repeatedly for each line generated by a dumper. This
callable can be configured using the first argument of a dumper's constructor,
but also using the
:method:`Symfony\\Component\\VarDumper\\Dumper\\AbstractDumper::setOutput`
method or the second argument of the
:method:`Symfony\\Component\\VarDumper\\Dumper\\AbstractDumper::dump` method.

For example, to get a dump as a string in a variable, you can do::

    use Symfony\Component\VarDumper\Cloner\VarCloner;
    use Symfony\Component\VarDumper\Dumper\CliDumper;

    $cloner = new VarCloner();
    $dumper = new CliDumper();
    $output = '';

    $dumper->dump(
        $cloner->cloneVar($variable),
        function ($line, $depth) use (&$output) {
            // A negative depth means "end of dump"
            if ($depth >= 0) {
                // Adds a two spaces indentation to the line
                $output .= str_repeat('  ', $depth).$line."\n";
            }
        }
    );

    // $output is now populated with the dump representation of $variable

Another option for doing the same could be::

    use Symfony\Component\VarDumper\Cloner\VarCloner;
    use Symfony\Component\VarDumper\Dumper\CliDumper;

    cloner = new VarCloner();
    $dumper = new CliDumper();
    $output = fopen('php://memory', 'r+b');

    $dumper->dump($cloner->cloneVar($variable), $output);
    rewind($output);
    $output = stream_get_contents($output);

    // $output is now populated with the dump representation of $variable

Dumpers implement the :class:`Symfony\\Component\\VarDumper\\Dumper\\DataDumperInterface`
interface that specifies the
:method:`dump(Data $data) <Symfony\\Component\\VarDumper\\Dumper\\DataDumperInterface::dump>`
method. They also typically implement the
:class:`Symfony\\Component\\VarDumper\\Cloner\\DumperInterface` that frees
them from re-implementing the logic required to walk through a
:class:`Symfony\\Component\\VarDumper\\Cloner\\Data` object's internal structure.

Casters
-------

Objects and resources nested in a PHP variable are "cast" to arrays in the
intermediate :class:`Symfony\\Component\\VarDumper\\Cloner\\Data`
representation. You can tweak the array representation for each object/resource
by hooking a Caster into this process. The component already includes many
casters for base PHP classes and other common classes.

If you want to build your own Caster, you can register one before cloning
a PHP variable. Casters are registered using either a Cloner's constructor
or its ``addCasters()`` method::

    use Symfony\Component\VarDumper\Cloner\VarCloner;

    $myCasters = array(...);
    $cloner = new VarCloner($myCasters);

    // or

    $cloner->addCasters($myCasters);

The provided ``$myCasters`` argument is an array that maps a class,
an interface or a resource type to a callable::

    $myCasters = array(
        'FooClass' => $myFooClassCallableCaster,
        ':bar resource' => $myBarResourceCallableCaster,
    );

As you can notice, resource types are prefixed by a ``:`` to prevent
colliding with a class name.

Because an object has one main class and potentially many parent classes
or interfaces, many casters can be applied to one object. In this case,
casters are called one after the other, starting from casters bound to the
interfaces, the parents classes and then the main class. Several casters
can also be registered for the same resource type/class/interface.
They are called in registration order.

Casters are responsible for returning the properties of the object or resource
being cloned in an array. They are callables that accept four arguments:

* the object or resource being casted,
* an array modelled for objects after PHP's native ``(array)`` cast operator,
* a :class:`Symfony\\Component\\VarDumper\\Cloner\\Stub` object
  representing the main properties of the object (class, type, etc.),
* true/false when the caster is called nested in a structure or not.

Here is a simple caster not doing anything::

    function myCaster($object, $array, $stub, $isNested)
    {
        // ... populate/alter $array to your needs

        return $array;
    }

For objects, the ``$array`` parameter comes pre-populated using PHP's native
``(array)`` casting operator or with the return value of ``$object->__debugInfo()``
if the magic method exists. Then, the return value of one Caster is given
as the array argument to the next Caster in the chain.

When casting with the ``(array)`` operator, PHP prefixes protected properties
with a ``\0*\0`` and private ones with the class owning the property. For example,
``\0Foobar\0`` will be the prefix for all private properties of objects of
type Foobar. Casters follow this convention and add two more prefixes: ``\0~\0``
is used for virtual properties and ``\0+\0`` for dynamic ones (runtime added
properties not in the class declaration).

.. note::

    Although you can, it is advised to not alter the state of an object
    while casting it in a Caster.

.. tip::

    Before writting your own casters, you should check the existing ones.
