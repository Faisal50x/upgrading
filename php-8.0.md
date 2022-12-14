PHP 8.0 UPGRADE NOTES

1. Backward Incompatible Changes
2. New Features
3. Changes in SAPI modules
4. Deprecated Functionality
5. Changed Functions
6. New Functions
7. New Classes and Interfaces
8. Removed Extensions and SAPIs
9. Other Changes to Extensions
10. New Global Constants
11. Changes to INI File Handling
12. Windows Support
13. Other Changes
14. Performance Improvements


========================================
1. Backward Incompatible Changes
========================================

- Core:
  . `match` is now a reserved keyword.
  . Assertion failures now throw by default. If the old behavior is desired,
    then set `assert.exception=0` in INI settings.
  . Methods with the same name as the class are no longer interpreted as
    constructors. The __construct() method should be used instead.
  . Removed ability to call non-static methods statically.
    Thus `is_callable` will fail when checking for a non-static method with a
    classname (must check with an object instance).
  . Removed (real) cast.
  . Removed (unset) cast.
  . Removed track_errors ini directive. This means that $php_errormsg is no
    longer available. The error_get_last() function may be used instead.
  . Removed the ability to define case-insensitive constants. The third
    argument to define() may no longer be true.
  . Access to undefined constants now always results in an Error exception.
    Previously, unqualified constant accesses resulted in a warning and were
    interpreted as strings.
  . Removed ability to specify an autoloader using an __autoload() function.
    spl_autoload_register() should be used instead.
  . Removed the $errcontext argument for custom error handlers.
  . Removed create_function(). Anonymous functions may be used instead.
  . Removed each(). foreach or ArrayIterator should be used instead.
  . Removed ability to unbind $this from closures that were created from a
    method, using Closure::fromCallable() or ReflectionMethod::getClosure().
  . Also removed ability to unbind $this from proper closures that contain uses
    of $this.
  . Removed ability to use array_key_exists() with objects. Use one of isset()
    or property_exists() instead.
  . Made the behavior of array_key_exists() regarding the type of the key
    parameter consistent with isset() and normal array access. All key types now
    use the usual coercions and array/object keys throw a TypeError.
  . Any array that has a number n as its first numeric key will use n+1 for
    its next implicit key, even if n is negative.
    RFC: https://wiki.php.net/rfc/negative_array_index
  . The default error_reporting level is now E_ALL. Previously it excluded
    E_NOTICE and E_DEPRECATED.
  . display_startup_errors is now enabled by default.
  . Using "parent" inside a class that has no parent will now result in a
    fatal compile-time error.
  . The @ operator will no longer silence fatal errors (E_ERROR, E_CORE_ERROR,
    E_COMPILE_ERROR, E_USER_ERROR, E_RECOVERABLE_ERROR, E_PARSE). Error handlers
    that expect error_reporting to be 0 when @ is used, should be adjusted to
    use a mask check instead:

        // Replace
        function my_error_handler($err_no, $err_msg, $filename, $linenum) {
            if (error_reporting() == 0) {
                return; // Silenced
            }
            // ...
        }

        // With
        function my_error_handler($err_no, $err_msg, $filename, $linenum) {
            if (!(error_reporting() & $err_no)) {
                return; // Silenced
            }
            // ...
        }

    Additionally, care should be taken that error messages are not displayed in
    production environments, which can result in information leaks. Please
    ensure that display_errors=Off is used in conjunction with error logging.
  . Following the hash comment operator # immediately with an opening bracket
    is not supported as a comment anymore since this syntax is now used for
    attributes.
    RFC: https://wiki.php.net/rfc/shorter_attribute_syntax_change
  . Inheritance errors due to incompatible method signatures (LSP violations)
    will now always generate a fatal error. Previously a warning was generated
    in some cases.
    RFC: https://wiki.php.net/rfc/lsp_errors
  . The precedence of the concatenation operator has changed relative to
    bitshifts and addition as well as subtraction.
    RFC: https://wiki.php.net/rfc/concatenation_precedence
  . Arguments with a default value that resolves to null at runtime will no
    longer implicitly mark the argument type as nullable. Either use an explicit
    nullable type, or an explicit null default value instead.

        // Replace
        function test(int $arg = CONST_RESOLVING_TO_NULL) {}
        // With
        function test(?int $arg = CONST_RESOLVING_TO_NULL) {}
        // Or
        function test(int $arg = null) {}
  . A number of warnings have been converted into Error exceptions:

    * Attempting to write to a property of a non-object. Previously this
      implicitly created an stdClass object for null, false and empty strings.
    * Attempting to append an element to an array for which the PHP_INT_MAX key
      is already used.
    * Attempting to use an invalid type (array or object) as an array key or
      string offset.
    * Attempting to write to an array index of a scalar value.
    * Attempting to unpack a non-array/Traversable.

    A number of notices have been converted into warnings:

    * Attempting to read an undefined variable.
    * Attempting to read an undefined property.
    * Attempting to read an undefined array key.
    * Attempting to read a property of a non-object.
    * Attempting to access an array index of a non-array.
    * Attempting to convert an array to string.
    * Attempting to use a resource as an array key.
    * Attempting to use null, a boolean, or a float as a string offset.
    * Attempting to read an out-of-bounds string offset.
    * Attempting to assign an empty string to a string offset.

    RFC: https://wiki.php.net/rfc/engine_warnings
  . Attempting to assign multiple bytes to a string offset will now emit a
    warning.
  . Unexpected characters in source files (such as null bytes outside of
    strings) will now result in a ParseError exception instead of a compile
    warning.
  . Uncaught exceptions now go through "clean shutdown", which means that
    destructors will be called after an uncaught exception.
  . Compile time fatal error "Only variables can be passed by reference" has
    been delayed until runtime and converted to "Argument cannot be passed by
    reference" exception.
  . Some "Only variables should be passed by reference" notices have been
    converted to "Argument cannot be passed by reference" exception.
  . The generated name for anonymous classes has changed. It will now include
    the name of the first parent or interface:

        new class extends ParentClass {};
        // -> ParentClass@anonymous
        new class implements FirstInterface, SecondInterface {};
        // -> FirstInterface@anonymous
        new class {};
        // -> class@anonymous

    The name shown above is still followed by a null byte and a unique
    suffix.
  . Non-absolute trait method references in trait alias adaptations are now
    required to be unambiguous:

        class X {
            use T1, T2 {
               func as otherFunc;
            }
            function func() {}
        }

    If both T1::func() and T2::func() exist, this code was previously silently
    accepted, and func was assumed to refer to T1::func. Now it will generate a
    fatal error instead, and either T1::func or T2::func needs to be written
    explicitly.
  . The signature of abstract methods defined in traits is now checked against
    the implementing class method:

        trait MyTrait {
            abstract private function neededByTrait(): string;
        }

        class MyClass {
            use MyTrait;

            // Error, because of return type mismatch.
            private function neededByTrait(): int { return 42; }
        }

    RFC: https://wiki.php.net/rfc/abstract_trait_method_validation
  . Disabled functions are now treated exactly like non-existent functions.
    Calling a disabled function will report it as unknown, and redefining a
    disabled function is now possible.
  . data: stream wrappers are no longer writable, which matches the documented
    behavior.
  . The arithmetic and bitwise operators
    +, -, *, /, **, %, <<, >>, &, |, ^, ~, ++, --
    will now consistently throw a TypeError when one of the operands is an
    array, resource or non-overloaded object. The only exception to this is the
    array + array merge operation, which remains supported.
    RFC: https://wiki.php.net/rfc/arithmetic_operator_type_checks
  . Float to string casting will now always behave locale-independently.
    RFC: https://wiki.php.net/rfc/locale_independent_float_to_string
  . Removed support for deprecated curly braces for offset access
    RFC: https://wiki.php.net/rfc/deprecate_curly_braces_array_access
  . Applying the final modifier on a private method will now produce a warning
    unless that method is the constructor.
    RFC: https://wiki.php.net/rfc/inheritance_private_methods
  . If an object constructor exit()s, the object destructor will no longer be
    called. This matches the behavior when the constructor throws.
  . Non-strict comparisons between numbers and non-numeric strings now work by
    casting the number to string and comparing the strings. Comparisons between
    numbers and numeric strings continue to work as before. Notably, this means
    that `0 == "not-a-number"` is considered false now.
    RFC: https://wiki.php.net/rfc/string_to_number_comparison
  . Namespaced names can no longer contain whitespace: While `Foo\Bar` will be
    recognized as a namespaced name, `Foo \ Bar` will not. Conversely, reserved
    keywords are now permitted as namespace segments, which may also change the
    interpretation of code: `new\x` is now the same as `constant('new\x')`, not
    `new \x()`.
    RFC: https://wiki.php.net/rfc/namespaced_names_as_token
  . Nested ternaries now require explicit parentheses.
    RFC: https://wiki.php.net/rfc/ternary_associativity
  . debug_backtrace() and Exception::getTrace() will no longer provide
    references to arguments. It will not be possible to change function
    arguments through the backtrace.
  . Numeric string handling has been altered to be more intuitive and less
    error-prone. Trailing whitespace is now allowed in numeric strings for
    consistency with how leading whitespace is treated. This mostly affects:
     - The is_numeric() function
     - String-to-string comparisons
     - Type declarations
     - Increment and decrement operations
    The concept of a "leading-numeric string" has been mostly dropped; the
    cases where this remains exist in order to ease migration. Strings which
    emitted an E_NOTICE "A non well-formed numeric value encountered" will now
    emit an E_WARNING "A non-numeric value encountered" and all strings which
    emitted an E_WARNING "A non-numeric value encountered" will now throw a
    TypeError. This mostly affects:
     - Arithmetic operations
     - Bitwise operations
    This E_WARNING to TypeError change also affects the E_WARNING
    "Illegal string offset 'string'" for illegal string offsets. The behavior
    of explicit casts to int/float from strings has not been changed.
    RFC: https://wiki.php.net/rfc/saner-numeric-strings
  . Magic Methods will now have their arguments and return types checked if
    they have them declared. The signatures should match the following list:

      __call(string $name, array $arguments): mixed
      __callStatic(string $name, array $arguments): mixed
      __clone(): void
      __debugInfo(): ?array
      __get(string $name): mixed
      __invoke(mixed $arguments): mixed
      __isset(string $name): bool
      __serialize(): array
      __set(string $name, mixed $value): void
      __set_state(array $properties): object
      __sleep(): array
      __unserialize(array $data): void
      __unset(string $name): void
      __wakeup(): void

    RFC: https://wiki.php.net/rfc/magic-methods-signature
  . call_user_func_array() array keys will now be interpreted as parameter names,
    instead of being silently ignored.

- COM:
  . Removed the ability to import case-insensitive constants from type
    libraries. The second argument to com_load_typelib() may no longer be false;
    com.autoregister_casesensitive may no longer be disabled; case-insensitive
    markers in com.typelib_file are ignored.

- Curl:
  . CURLOPT_POSTFIELDS no longer accepts objects as arrays. To interpret an
    object as an array, perform an explicit (array) cast. The same applies to
    other options accepting arrays as well.

- Date:
  . mktime() and gmmktime() now require at least one argument. time() can be
    used to get the current timestamp.

- dom:
  . Remove unimplemented classes from ext/dom that had no behavior and contained
    test data. These classes have also been removed in the latest version of DOM
    standard:

     * DOMNameList
     * DomImplementationList
     * DOMConfiguration
     * DomError
     * DomErrorHandler
     * DOMImplementationSource
     * DOMLocator
     * DOMUserDataHandler
     * DOMTypeInfo

- Enchant:
  . enchant_broker_list_dicts(), enchant_broker_describe() and
    enchant_dict_suggest() will now return an empty array instead of null.
  . enchant_broker_init() will now return an EnchantBroker object rather than
    a resource. Return value checks using is_resource() should be replaced with
    checks for `false`.
  . enchant_broker_request_dict() and enchant_broker_request_pwl_dict() will now
    return an EnchantDictionary object rather than a resource. Return value
    checks using is_resource() should be replaced with checks for `false`.

- Exif:
  . Removed read_exif_data(). exif_read_data() should be used instead.

- Filter:
  . The FILTER_FLAG_SCHEME_REQUIRED and FILTER_FLAG_HOST_REQUIRED flags for the
    FILTER_VALIDATE_URL filter have been removed. The scheme and host are (and
    have been) always required.
  . The INPUT_REQUEST and INPUT_SESSION source for filter_input() etc have been
    removed. These were never implemented and their use always generated a
    warning.

- GD:
  . The GD extension now uses a GdImage objects as the underlying data structure
    for images, rather than resources. These objects are completely opaque, i.e.
    they don't have any methods. Return value checks using is_resource() should
    be replaced with checks for `false`. The imagedestroy() function no longer
    has an effect, instead the GdImage instance is automatically destroyed if
    it is no longer referenced.
  . The deprecated function image2wbmp() has been removed.
    RFC: https://wiki.php.net/rfc/image2wbmp
  . The deprecated functions png2wbmp() and jpeg2wbmp() have been removed.
    RFC: https://wiki.php.net/rfc/deprecate-png-jpeg-2wbmp
  . The default $mode parameter of imagecropauto() no longer accepts -1.
    IMG_CROP_DEFAULT should be used instead.

- GMP:
  . gmp_random() has been removed. One of gmp_random_range() or
    gmp_random_bits() should be used instead.

- Iconv:
  . iconv() implementations which do not properly set errno in case of errors
    are no longer supported.

- IMAP:
  . The unused default_host argument of imap_headerinfo() has been removed.
  . The imap_header() function which is an alias of imap_headerinfo() has been removed.

- Intl:
  . The deprecated constant INTL_IDNA_VARIANT_2003 has been removed.
    RFC: https://wiki.php.net/rfc/deprecate-and-remove-intl_idna_variant_2003
  . The deprecated Normalizer::NONE constant has been removed.
  . The IntlDateFormatter::RELATIVE_FULL, IntlDateFormatter::RELATIVE_LONG,
    IntlDateFormatter::RELATIVE_MEDIUM, and IntlDateFormatter::RELATIVE_SHORT
    constants have been added.

- LDAP:
  . The deprecated function ldap_sort has been removed.
  . The deprecated function ldap_control_paged_result has been removed.
  . The deprecated function ldap_control_paged_result_response has been removed.
  . The interface of ldap_set_rebind_proc has changed; the $callback parameter
    does not accept empty string anymore; null value shall be used instead.

- Mbstring:
  . The mbstring.func_overload directive has been removed. The related
    MB_OVERLOAD_MAIL, MB_OVERLOAD_STRING, and MB_OVERLOAD_REGEX constants have
    also been removed. Finally, the "func_overload" and "func_overload_list"
    entries in mb_get_info() have been removed.
  . mb_parse_str() can no longer be used without specifying a result array.
  . A number of deprecated mbregex aliases have been removed. See the following
    list for which functions should be used instead:

     * mbregex_encoding()      -> mb_regex_encoding()
     * mbereg()                -> mb_ereg()
     * mberegi()               -> mb_eregi()
     * mbereg_replace()        -> mb_ereg_replace()
     * mberegi_replace()       -> mb_eregi_replace()
     * mbsplit()               -> mb_split()
     * mbereg_match()          -> mb_ereg_match()
     * mbereg_search()         -> mb_ereg_search()
     * mbereg_search_pos()     -> mb_ereg_search_pos()
     * mbereg_search_regs()    -> mb_ereg_search_regs()
     * mbereg_search_init()    -> mb_ereg_search_init()
     * mbereg_search_getregs() -> mb_ereg_search_getregs()
     * mbereg_search_getpos()  -> mb_ereg_search_getpos()
     * mbereg_search_setpos()  -> mb_ereg_search_setpos()

  . The 'e' modifier for mb_ereg_replace() has been removed.
    mb_ereg_replace_callback() should be used instead.
  . A non-string pattern argument to mb_ereg_replace() will now be interpreted
    as a string instead of an ASCII codepoint. The previous behavior may be
    restored with an explicit call to chr().
  . The needle argument for mb_strpos(), mb_strrpos(), mb_stripos(),
    mb_strripos(), mb_strstr(), mb_stristr(), mb_strrchr() and mb_strrichr() can
    now be empty.
  . The $is_hex parameter, which was not used internally, has been removed from
    mb_decode_numericentity().
  . The legacy behavior of passing the encoding as the third argument instead
    of an offset for the mb_strrpos() function has been removed; provide an
    explicit 0 offset with the encoding as the fourth argument instead.
  . The ISO_8859-* character encoding aliases have been replaced by ISO8859-*
    aliases for better interoperability with the iconv extension. The mbregex
    ISO 8859 aliases with underscores (ISO_8859_* and ISO8859_*) have also been
    removed.
  . mb_ereg() and mb_eregi() will now return boolean true on a successful
    match. Previously they returned integer 1 if $matches was not passed, or
    max(1, strlen($reg[0])) is $matches was passed.

- OCI8:
  . The OCI-Lob class is now called OCILob, and the OCI-Collection class is now
    called OCICollection for name compliance enforced by PHP 8 arginfo
    type annotation tooling.
  . Several alias functions have been marked as deprecated.
  . oci_internal_debug() and its alias ociinternaldebug() have been removed.

- ODBC:
  . odbc_connect() no longer reuses persistent connections.
  . The unused flags parameter of odbc_exec() has been removed.

- OpenSSL:
  . openssl_x509_read() and openssl_csr_sign() will now return an
    OpenSSLCertificate object rather than a resource. Return value checks using
    is_resource() should be replaced with checks for `false`.
  . The openssl_x509_free() function is deprecated and no longer has an effect,
    instead the OpenSSLCertificate instance is automatically destroyed if it is no
    longer referenced.
  . openssl_csr_new() will now return an OpenSSLCertificateSigningRequest object
    rather than a resource. Return value checks using is_resource() should be
    replaced with checks for `false`.
  . openssl_pkey_new() will now return an OpenSSLAsymmetricKey object rather than a
    resource. Return value checks using is_resource() should be replaced with
    checks for `false`.
  . The openssl_pkey_free() function is deprecated and no longer has an effect,
    instead the OpenSSLAsymmetricKey instance is automatically destroyed if it is no
    longer referenced.
  . openssl_seal() and openssl_open() now require $method to be passed, as the
    previous default of "RC4" is considered insecure.

- PCRE:
  . When passing invalid escape sequences they are no longer interpreted as
    literals. This behavior previously required the X modifier - which is
    now ignored.

- PDO:
  . The default error handling mode has been changed from "silent" to
    "exceptions". See https://www.php.net/manual/en/pdo.error-handling.php
    for details of behavior changes and how to explicitly set this attribute.
    RFC: https://wiki.php.net/rfc/pdo_default_errmode
  . The signatures of some PDO methods have changed:

        PDO::query(
            string $statement, ?int $fetch_mode = null, ...$fetch_mode_args)
        PDOStatement::setFetchMode(int $mode, ...$params)

- PDO MySQL:
  . PDO::inTransaction() now reports the actual transaction state of the
    connection, rather than an approximation maintained by PDO. If a query that
    is subject to "implicit commit" is executed, PDO::inTransaction() will
    subsequently return false, as a transaction is no longer active.

- PDO_ODBC:
  . The php.ini directive pdo_odbc.db2_instance_name has been removed

- pgsql:
  . The deprecated pg_connect() syntax using multiple parameters instead of a
    connection string is no longer supported.
  . The deprecated pg_lo_import() and pg_lo_export() signature that passes the
    connection as the last argument is no longer supported. The connection
    should be passed as first argument instead.
  . pg_fetch_all() will now return an empty array instead of false for result
    sets with zero rows.

- Phar:
  . Metadata associated with a phar will no longer be automatically unserialized,
    to fix potential security vulnerabilities due to object instantiation, autoloading, etc.
    RFC: https://wiki.php.net/rfc/phar_stop_autoloading_metadata

- Reflection:
  . The method signatures

        ReflectionClass::newInstance($args)
        ReflectionFunction::invoke($args)
        ReflectionMethod::invoke($object, $args)

    have been changed to:

        ReflectionClass::newInstance(...$args)
        ReflectionFunction::invoke(...$args)
        ReflectionMethod::invoke($object, ...$args)

    Code that must be compatible with both PHP 7 and PHP 8 can use the following
    signatures to be compatible with both versions:

        ReflectionClass::newInstance($arg = null, ...$args)
        ReflectionFunction::invoke($arg = null, ...$args)
        ReflectionMethod::invoke($object, $arg = null, ...$args)

  . The ReflectionType::__toString() method will now return a complete debug
    representation of the type, and is no longer deprecated. In particular the
    result will include a nullability indicator for nullable types. The format
    of the return value is not stable and may change between PHP versions.
  . Reflection export() methods have been removed.
  . The following methods can now return information about default values of
    parameters of internal functions:
        ReflectionParameter::isDefaultValueAvailable()
        ReflectionParameter::getDefaultValue()
        ReflectionParameter::isDefaultValueConstant()
        ReflectionParameter::getDefaultValueConstantName()
  . ReflectionMethod::isConstructor() and ReflectionMethod::isDestructor() now
    also return true for `__construct` and `__destruct` methods of interfaces.
    Previously, this would only be true for methods of classes and traits.
  . ReflectionType::isBuiltin() method has been moved to ReflectionNamedType.
    ReflectionUnionType does not have it.

- Sockets:
  . The deprecated AI_IDN_ALLOW_UNASSIGNED and AI_IDN_USE_STD3_ASCII_RULES
    flags for socket_addrinfo_lookup() have been removed.
  . socket_create(), socket_create_listen(), socket_accept(),
    socket_import_stream(), socket_addrinfo_connect(), socket_addrinfo_bind(),
    and socket_wsaprotocol_info_import() will now return a Socket object rather
    than a resource. Return value checks using is_resource() should be replaced
    with checks for `false`.
  . socket_addrinfo_lookup() will now return an array of AddressInfo objects
    rather than resources.

- SPL:
  . SplFileObject::fgetss() has been removed.
  . SplFileObject::seek() now always seeks to the beginning of the line.
    Previously, positions >=1 sought to the beginning of the next line.
  . SplHeap::compare($a, $b) now specifies a method signature. Inheriting
    classes implementing this method will now have to use a compatible
    method signature.
  . SplDoublyLinkedList::push() now returns void instead of true
  . SplDoublyLinkedList::unshift() now returns void instead of true
  . SplQueue::enqueue() now returns void instead of true
  . spl_autoload_register() will now always throw a TypeError on invalid
    arguments, therefore the second argument $do_throw is ignored and a
    notice will be emitted if it is set to false.
  . SplFixedArray is now an IteratorAggregate and not an Iterator.
    SplFixedArray::rewind(), ::current(), ::key(), ::next(), and ::valid()
    have been removed. In their place, SplFixedArray::getIterator() has been
    added. Any code which uses explicit iteration over SplFixedArray must now
    obtain an Iterator through SplFixedArray::getIterator(). This means that
    SplFixedArray is now safe to use in nested loops.

- Standard:
  . assert() will no longer evaluate string arguments, instead they will be
    treated like any other argument. assert($a == $b) should be used instead of
    assert('$a == $b'). The assert.quiet_eval ini directive and
    ASSERT_QUIET_EVAL constants have also been removed, as they would no longer
    have any effect.
  . parse_str() can no longer be used without specifying a result array.
  . fgetss() has been removed.
  . The string.strip_tags filter has been removed.
  . The needle argument of strpos(), strrpos(), stripos(), strripos(), strstr(),
    strchr(), strrchr(), and stristr() will now always be interpreted as a
    string. Previously non-string needles were interpreted as an ASCII code
    point. An explicit call to chr() can be used to restore the previous
    behavior.
  . The needle argument for strpos(), strrpos(), stripos(), strripos(),
    strstr(), stristr() and strrchr() can now be empty.
  . The length argument for substr(), substr_count(), substr_compare(), and
    iconv_substr() can now be null. Null values will behave as if no length
    argument was provided and will therefore return the remainder of the string
    instead of an empty string.
  . The length argument for array_splice() can now be null. Null values will
    behave identically to omitting the argument, thus removing everything from
    the 'offset' to the end of the array.
  . The args argument of vsprintf(), vfprintf(), and vprintf() must now be an
    array. Previously any type was accepted.
  . The 'salt' option of password_hash() is no longer supported. If the 'salt'
    option is used a warning is generated, the provided salt is ignored, and a
    generated salt is used instead.
  . The quotemeta() function will now return an empty string if an empty string
    was passed. Previously false was returned.
  . hebrevc() has been removed.
  . convert_cyr_string() has been removed.
  . money_format() has been removed.
  . ezmlm_hash() has been removed.
  . restore_include_path() has been removed.
  . get_magic_quotes_gpc() and get_magic_quotes_runtime() has been removed.
  . FILTER_SANITIZE_MAGIC_QUOTES has been removed.
  . Calling implode() with parameters in a reverse order ($pieces, $glue) is no
    longer supported.
  . parse_url() will now distinguish absent and empty queries and fragments:

        http://example.com/foo   => query = null, fragment = null
        http://example.com/foo?  => query = "",   fragment = null
        http://example.com/foo#  => query = null, fragment = ""
        http://example.com/foo?# => query = "",   fragment = ""

    Previously all cases resulted in query and fragment being null.
  . var_dump() and debug_zval_dump() will now print floating-point numbers
    using serialize_precision rather than precision. In a default configuration,
    this means that floating-point numbers are now printed with full accuracy
    by these debugging functions.
  . If the array returned by __sleep() contains non-existing properties, these
    are now silently ignored. Previously, such properties would have been
    serialized as if they had the value NULL.
  . The default locale on startup is now always "C". No locales are inherited
    from the environment by default. Previously, LC_ALL was set to "C", while
    LC_CTYPE was inherited from the environment. However, some functions did not
    respect the inherited locale without an explicit setlocale() call. An
    explicit setlocale() call is now always required if you wish to change any
    locale component from the default.
  . Removed deprecated DES fallback in crypt(). If an unknown salt format is
    passed to crypt(), the function will fail with *0 instead of falling back
    to a weak DES hash now.
  . Specifying out of range rounds for sha256/sha512 crypt() will now fail with
    *0 instead of clamping to the closest limit. This matches glibc behavior.
  . The result of sorting functions may have changed, if the array contains
    elements that compare as equal.
  . Sort comparison functions that return true or false will now throw a
    deprecation warning, and should be replaced with an implementation
    that returns an integer less than, equal to, or greater than zero.

        // Replace
        usort($array, fn($a, $b) => $a > $b);
        // With
        usort($array, fn($a, $b) => $a <=> $b);

  . Any functions accepting callbacks that are not explicitly specified to
    accept parameters by reference will now warn if a callback with reference
    parameters is used. Examples include array_filter() and array_reduce().
    This was already the case for most, but not all, functions previously.
  . The HTTP stream wrapper as used by functions like file_get_contents()
    now advertises HTTP/1.1 rather than HTTP/1.0 by default. This does not
    change the behavior of the client, but may cause servers to respond
    differently. To retain the old behavior, set the 'protocol_version'
    stream context option, e.g.

        $ctx = stream_context_create(['http' => ['protocol_version' => '1.0']]);
        echo file_get_contents('http://example.org', false, $ctx);
  . Calling crypt() without an explicit salt is no longer supported. If you
    would like to produce a strong hash with an auto-generated salt, use
    password_hash() instead.
  . substr(), mb_substr(), iconv_substr() and grapheme_substr() now consistently
    clamp out-of-bounds offsets to the string boundary. Previously, false was
    returned instead of the empty string in some cases.
  . Populating $http_response_header variable by the HTTP stream wrapper
    doesn't force rebuilding of symbol table anymore.

- Sysvmsg:
  . msg_get_queue() will now return an SysvMessageQueue object rather than a
    resource. Return value checks using is_resource() should be replaced with
    checks for `false`.

- Sysvsem:
  . sem_get() will now return an SysvSemaphore object rather than a resource.
    Return value checks using is_resource() should be replaced with checks
    for `false`.
  . The $auto_release parameter of sem_get() was changed to accept bool values
    rather than int.

- Sysvshm:
  . shm_attach() will now return an SysvSharedMemory object rather than a resource.
    Return value checks using is_resource() should be replaced with checks
    for `false`.

- tidy:
  . The $use_include_path parameter, which was not used internally, has been
    removed from tidy_repair_string().
  . tidy::repairString() and tidy::repairFile() became static methods

- Tokenizer:
  . T_COMMENT tokens will no longer include a trailing newline. The newline will
    instead be part of a following T_WHITESPACE token. It should be noted that
    T_COMMENT is not always followed by whitespace, it may also be followed by
    T_CLOSE_TAG or end-of-file.
  . Namespaced names are now represented using the T_NAME_QUALIFIED (Foo\Bar),
    T_NAME_FULLY_QUALIFIED (\Foo\Bar) and T_NAME_RELATIVE (namespace\Foo\Bar)
    tokens. T_NS_SEPARATOR is only used for standalone namespace separators,
    and only syntactially valid in conjunction with group use declarations.
    RFC: https://wiki.php.net/rfc/namespaced_names_as_token

- XML:
  . xml_parser_create(_ns) will now return an XMLParser object rather than a
    resource. Return value checks using is_resource() should be replaced with
    checks for `false`. The xml_parser_free() function no longer has an effect,
    instead the XMLParser instance is automatically destroyed if it is no longer
    referenced.

- XMLReader:
  . XMLReader::open() and XMLReader::xml() are now static methods. They still
    can be called dynamically, though, but inheriting classes need to declare
    them as static if they override these methods.

- XMLWriter:
  . The XMLWriter functions now accept and return, respectively, XMLWriter
    objects instead of resources.

- Zip:
  . ZipArchive::OPSYS_Z_CPM has been removed (this name was a typo). Use
    ZipArchive::OPSYS_CPM instead.

- Zlib:
  . gzgetss() has been removed.
  . inflate_init() will now return an InflateContext object rather than a
    resource. Return value checks using is_resource() should be replaced with
    checks for `false`.
  . deflate_init() will now return a DeflateContext object rather than a
    resource. Return value checks using is_resource() should be replaced with
    checks for `false`.
  . zlib.output_compression is no longer automatically disabled for
    Content-Type: image/*.

========================================
2. New Features
========================================

- Core:
  . Added support for union types.
    RFC: https://wiki.php.net/rfc/union_types_v2
  . Added WeakMap.
    RFC: https://wiki.php.net/rfc/weak_maps
  . Added ValueError class.
  . Any number of function parameters may now be replaced by a variadic
    argument, as long as the types are compatible. For example, the following
    code is now allowed:

        class A {
            public function method(int $many, string $parameters, $here) {}
        }
        class B extends A {
            public function method(...$everything) {}
        }
  . "static" (as in "late static binding") can now be used as a return type:

        class Test {
            public function create(): static {
                return new static();
            }
        }

    RFC: https://wiki.php.net/rfc/static_return_type
  . It is now possible to fetch the class name of an object using
    `$object::class`. The result is the same as `get_class($object)`.
    RFC: https://wiki.php.net/rfc/class_name_literal_on_object
  . New and instanceof can now be used with arbitrary expressions, using
    `new (expression)(...$args)` and `$obj instanceof (expression)`.
    RFC: https://wiki.php.net/rfc/variable_syntax_tweaks
  . Some consistency fixes to variable syntax have been applied, for example
    writing `Foo::BAR::$baz` is now allowed.
    RFC: https://wiki.php.net/rfc/variable_syntax_tweaks
  . Added Stringable interface, which is automatically implemented if a class
    defines a __toString() method.
    RFC: https://wiki.php.net/rfc/stringable
  . Traits can now define abstract private methods.
    RFC: https://wiki.php.net/rfc/abstract_trait_method_validation
  . `throw` can now be used as an expression.
    RFC: https://wiki.php.net/rfc/throw_expression
  . An optional trailing comma is now allowed in parameter lists.
    RFC: https://wiki.php.net/rfc/trailing_comma_in_parameter_list
  . It is now possible to write `catch (Exception)` to catch an exception
    without storing it in a variable.
    RFC: https://wiki.php.net/rfc/non-capturing_catches
  . Added support for mixed type
    RFC: https://wiki.php.net/rfc/mixed_type_v2
  . Added support for Attributes
    RFC: https://wiki.php.net/rfc/attributes_v2
    RFC: https://wiki.php.net/rfc/attribute_amendments
    RFC: https://wiki.php.net/rfc/shorter_attribute_syntax
    RFC: https://wiki.php.net/rfc/shorter_attribute_syntax_change
  . Added support for constructor property promotion (declaring properties in
    the constructor signature).
    RFC: https://wiki.php.net/rfc/constructor_promotion
  . Added support for `match` expression.
    RFC: https://wiki.php.net/rfc/match_expression_v2
  . Private methods declared on a parent class no longer enforce any
    inheritance rules on the methods of a child class. (with the exception of
    final private constructors)
    RFC: https://wiki.php.net/rfc/inheritance_private_methods
  . Added support for nullsafe operator (`?->`).
    RFC: https://wiki.php.net/rfc/nullsafe_operator
  . Added support for named arguments.
    RFC: https://wiki.php.net/rfc/named_params

- Date:
  . Added DateTime::createFromInterface() and
    DateTimeImmutable::createFromInterface().
  . Added the DateTime format specifier "p" which is the same as "P" but
    returning "Z" for UTC.

- Dom:
  . Introduce DOMParentNode and DOMChildNode with new traversal and
    manipulation APIs.
    RFC: https://wiki.php.net/rfc/dom_living_standard_api

- Enchant:
  . enchant_dict_add()
  . enchant_dict_is_added()
  . LIBENCHANT_VERSION macro

- FPM:
  . Added a new option pm.status_listen that allows getting status from
    different endpoint (e.g. port or UDS file) which is useful for getting
    status when all children are busy with serving long running requests.

- Hash:
  . HashContext objects can now be serialized.

- Opcache:
  . If the opcache.record_warnings ini setting is enabled, opcache will record
    compile-time warnings and replay them on the next include, even if it is
    served from cache.

- OpenSSL:
  . Added Cryptographic Message Syntax (CMS) (RFC 5652) support composed of
    functions for encryption, decryption, signing, verifying and reading. The
    API is similar to the API for PKCS #7 functions with an addition of new
    encoding constants: OPENSSL_ENCODING_DER, OPENSSL_ENCODING_SMIME and
    OPENSSL_ENCODING_PEM.

- Standard:
  . printf() and friends now support the %h and %H format specifiers. These
    are the same as %g and %G, but always use "." as the decimal separator,
    rather than determining it through the LC_NUMERIC locale.
  . printf() and friends now support using "*" as width or precision, in which
    case the width/precision is passed as an argument to printf. This also
    allows using precision -1 with %g, %G, %h and %H. For example, the following
    code can be used to reproduce PHP's default floating point formatting:

        printf("%.*H", (int) ini_get("precision"), $float);
        printf("%.*H", (int) ini_get("serialize_precision"), $float);

  . proc_open() now supports pseudo-terminal (PTY) descriptors. The following
    attaches stdin, stdout and stderr to the same PTY:

        $proc = proc_open($command, [['pty'], ['pty'], ['pty']], $pipes);

  . proc_open() now supports socket pair descriptors. The following attaches
    a distinct socket pair to stdin, stdout and stderr:

        $proc = proc_open(
            $command, [['socket'], ['socket'], ['socket']], $pipes);

    Unlike pipes, sockets do not suffer from blocking I/O issues on Windows.
    However, not all programs may work correctly with stdio sockets.
  . Sorting functions are now stable, which means that equal-comparing elements
    will retain their original order.
    RFC: https://wiki.php.net/rfc/stable_sorting
  . array_diff(), array_intersect() and their variations can now be used with
    a single array as argument. This means that usages like the following are
    now possible:

        // OK even if $excludes is empty.
        array_diff($array, ...$excludes);
        // OK even if $arrays only contains a single array.
        array_intersect(...$arrays);
  . The $flag parameter of ob_implicit_flush() was changed to accept bool
    values rather than int.

- Zip:
  . Extension updated to version 1.19.1
  . New ZipArchive::lastId property to get index value of last added entry.
  . Error can be checked after an archive is closed using ZipArchive::status,
    ZipArchive::statusSys properties or ZipArchive::getStatusString() method.
  . The remove_path option of ZipArchive::addGlob() and ::addPattern() is now
    treated as arbitrary string prefix (for consistency with the add_path
    option), whereas formerly it was treated as directory name.
  . Optional compression / encryption features are listed in phpinfo.

========================================
3. Changes in SAPI modules
========================================

- Apache:
  . The PHP module has been renamed from php7_module to php_module.

========================================
4. Deprecated Functionality
========================================

- Core:
  . Declaring a required parameter after an optional one is deprecated. As an
    exception, declaring a parameter of the form "Type $param = null" before
    a required one continues to be allowed, because this pattern was sometimes
    used to achieve nullable types in older PHP versions.

        function test($a = [], $b) {}       // Deprecated
        function test(Foo $a = null, $b) {} // Allowed
  . Calling get_defined_functions() with $exclude_disabled explicitly set to
    false is deprecated. get_defined_functions() will never include disabled
    functions.

- Enchant:
  . enchant_broker_set_dict_path and enchant_broker_get_dict_path
    not available in libenchant < 1.5 nor in libenchant-2
  . enchant_dict_add_to_personal, use enchant_dict_add instead
  . enchant_dict_is_in_session, use enchant_dict_is_added instead
  . enchant_broker_free and enchant_broker_free_dict, unset the object instead
  . ENCHANT_MYSPELL and ENCHANT_ISPELL constants

- LibXML:
  . libxml_disable_entity_loader() has been deprecated. As libxml 2.9.0 is now
    required, external entity loading is guaranteed to be disabled by default,
    and this function is no longer needed to protect against XXE attacks.

- PGSQL / PDO PGSQL:
  . The constant PGSQL_LIBPQ_VERSION_STR has now the same value as
    PGSQL_LIBPQ_VERSION, and thus is deprecated.
  . Function aliases in the pgsql extension have been deprecated.

- Zip:
  . Using empty file as ZipArchive is deprecated. Libzip 1.6.0
    do not accept empty files as valid zip archives any longer.
    Existing workaround will be removed in next version.
  . The procedural API of Zip is deprecated. Use ZipArchive instead.

- Reflection:
  . ReflectionFunction::isDisabled() is deprecated, as it is no longer possible
    to create a ReflectionFunction for a disabled function. This method now
    always returns false.
  . ReflectionParameter::getClass(), ReflectionParameter::isArray(), and
    ReflectionParameter::isCallable() are deprecated.
    ReflectionParameter::getType() and the ReflectionType APIs should be used
    instead.

========================================
5. Changed Functions
========================================

- Reflection:
  . ReflectionClass::getConstants and ReflectionClass::getReflectionConstants results
    can be now filtered via a new parameter `$filter`. 3 new constants were added to
    be used with it:

      ReflectionClassConstant::IS_PUBLIC
      ReflectionClassConstant::IS_PROTECTED
      ReflectionClassConstant::IS_PRIVATE
      

- Zip
 . ZipArchive::addGlob and ZipArchive::addPattern methods accept more
   values in the "options" array argument:
   . flags
   . comp_method
   . comp_flags
   . env_method
   . enc_password
 . ZipArchive::addEmptyDir, ZipArchive::addFile and aZipArchive::addFromString
   methods have a new "flags" argument. This allows managing name encoding
   (ZipArchive::FL_ENC_*) and entry replacement (ZipArchive::FL_OVERWRITE)
 . ZipArchive::extractTo now restore file modification time.

========================================
6. New Functions
========================================

- Core:
  . Added get_resource_id($resource) function, which returns the same value as
    (int) $resource. It provides the same functionality under a clearer API.

- LDAP:
  . Added ldap_count_references(), which returns the number of reference
    messages in a search result.

- OpenSSL:
  . Added openssl_cms_encrypt() encrypts the message in the file with the
    certificates and outputs the result to the supplied file.
  . Added openssl_cms_decrypt() that decrypts the S/MIME message in the file
    and outputs the results to the supplied file.
  . Added openssl_cms_read() that exports the CMS file to an array of PEM
    certificates.
  . Added openssl_cms_sign() that signs the MIME message in the file with
    a cert and key and output the result to the supplied file.
  . Added openssl_cms_verify() that verifies that the data block is intact,
    the signer is who they say they are, and returns the certs of the signers.

- PCRE:
  . Added preg_last_error_msg(), which returns a human-readable message for
    the last PCRE error. It complements preg_last_error(), which returns an
    integer enum instead.

- SQLite3:
  . Add SQLite3::setAuthorizer() and respective class constants to set a
    userland callback that will be used to authorize or not an action on the
    database.
    PR: https://github.com/php/php-src/pull/4797

- Standard:
  . Added

        str_contains(string $haystack, string $needle): bool
        str_starts_with(string $haystack, string $needle): bool
        str_ends_with(string $haystack, string $needle): bool

    functions, which check whether $haystack contains, starts with or ends with
    $needle.
    RFC: https://wiki.php.net/rfc/str_contains
    RFC: https://wiki.php.net/rfc/add_str_starts_with_and_ends_with_functions
  . Added fdiv() function, which performs a floating-point division under
    IEEE 754 semantics. Division by zero is considered well-defined and
    will return one of Inf, -Inf or NaN.
  . Added get_debug_type() function, which returns a type useful for error
    messages. Unlike gettype(), it uses canonical type names, returns class
    names for objects, and indicates the resource type for resources.
    RFC: https://wiki.php.net/rfc/get_debug_type

- Zip:
  . ZipArchive::setMtimeName and ZipArchive::setMtimeIndex to set the
    modification time of an entry.
  . ZipArchive::registerProgressCallback to provide updates during archive close.
  . ZipArchive::registerCancelCallback to allow cancellation during archive close.
  . ZipArchive::replaceFile to replace an entry content.
  . ZipArchive::isCompressionMethodSupported to check optional compression
    features.
  . ZipArchive::isEncryptionMethodSupported to check optional encryption
    features.

========================================
7. New Classes and Interfaces
========================================

- Tokenizer:
  . The new PhpToken class adds an object-based interface to the tokenizer.
    It provides a more uniform and ergonomic representation, while being more
    memory efficient and faster.
    RFC: https://wiki.php.net/rfc/token_as_object

========================================
8. Removed Extensions and SAPIs
========================================

- XML-RPC:
  . The xmlrpc extension has been unbundled and moved to PECL.
    RFC: https://wiki.php.net/rfc/unbundle_xmlprc

========================================
9. Other Changes to Extensions
========================================

- CURL:
  . The CURL extension now requires at least libcurl 7.29.0.
  . curl_init() will now return a CurlHandle object rather than a resource.
    Return value checks using is_resource() should be replaced with
    checks for `false`. The curl_close() function no longer has an effect,
    instead the CurlHandle instance is automatically destroyed if it is no
    longer referenced.
  . curl_multi_init() will now return a CurlMultiHandle object rather than a
    resource. Return value checks using is_resource() should be replaced with
    checks for `false`. The curl_multi_close() function no longer has an effect,
    instead the CurlMultiHandle instance is automatically destroyed if it is no
    longer referenced.
  . curl_share_init() will now return a CurlShareHandle object rather than a
    resource. Return value checks using is_resource() should be replaced with
    checks for `false`. The curl_share_close() function no longer has an effect,
    instead the CurlShareHandle instance is automatically destroyed if it is no
    longer referenced.
  . The deprecated parameter `$version` of curl_version() has been removed.

- Date:
  . DatePeriod now implements IteratorAggregate (instead of Traversable).

- DOM:
  . DOMNamedNodeMap now implements IteratorAggregate (instead of Traversable).
  . DOMNodeList now implements IteratorAggregate (instead of Traversable).

- Intl:
  . IntlBreakIterator now implements IteratorAggregate (instead of Traversable).
  . ResourceBundle now implements IteratorAggregate (instead of Traversable).

- Enchant:
  . The enchant extension now uses libenchant-2 by default when available.
    libenchant version 1 is still supported but is deprecated and could
    be removed in the future.

- GD:
  . The $num_points parameter of imagepolygon(), imageopenpolygon() and
    imagefilledpolygon() is now optional, i.e. these functions may be called
    with either 3 or 4 arguments. If the arguments is omitted, it is calculated
    as count($points)/2.
  . The function imagegetinterpolation() to get the current interpolation method
    has been added.

- JSON:
  . The JSON extension cannot be disabled anymore and is always an integral part
    of any PHP build, similar to the date extension.

- MBString:
  . The Unicode data tables have been updated to version 13.0.0.

- PDO:
  . PDOStatement now implements IteratorAggregate (instead of Traversable).

- LibXML:
  . The minimum required libxml version is now 2.9.0. This means that external
    entity loading is now guaranteed to be disabled by default, and no extra
    steps need to be taken to protect against XXE attacks.

- MySQLi / PDO MySQL:
  . When mysqlnd is not used (which is the default and recommended option),
    the minimum supported libmysqlclient version is now 5.5.
  . mysqli_result now implements IteratorAggregate (instead of Traversable).

- PGSQL / PDO PGSQL:
  . The PGSQL and PDO PGSQL extensions now require at least libpq 9.1.

- Readline:
  . Calling readline_completion_function() before the interactive prompt starts
    (e.g. in auto_prepend_file) will now override the default interactive prompt
	completion function. Previously, readline_completion_function() only worked
	when called after starting the interactive prompt.

- SimpleXML:
  . SimpleXMLElement now implements RecursiveIterator and absorbed the
    functionality of SimpleXMLIterator. SimpleXMLIterator is an empty extension
    of SimpleXMLElement.

- Shmop:
  . shmop_open() will now return a Shmop object rather than a resource. Return
    value checks using is_resource() should be replaced with checks for `false`.
    The shmop_close() function no longer has an effect, and is deprecated;
    instead the Shmop instance is automatically destroyed if it is no longer
    referenced.

========================================
10. New Global Constants
========================================

- Filter:
  . FILTER_VALIDATE_BOOL has been added as an alias for FILTER_VALIDATE_BOOLEAN.
    The new name is preferred, as it uses the canonical type name.

========================================
11. Changes to INI File Handling
========================================

- zend.exception_string_param_max_len
  . New INI directive to set the maximum string length in an argument of a
    stringified stack strace.

- com.dotnet_version
  . New INI directive to choose the version of the .NET framework to use for
    dotnet objects.

========================================
12. Windows Support
========================================

- Standard:
  . Program execution functions (proc_open(), exec(), popen() etc.) using the
    shell now consistently execute `%comspec% /s /c "$commandline"`, which has
    the same effect as executing `$commandline` (without additional quotes).

- GD:
  . php_gd2.dll has been renamed to php_gd.dll.

- php-test-pack:
  . The test runner has been renamed from run-test.php to run-tests.php, to
    match its name in php-src.

========================================
13. Other Changes
========================================

- EBCDIC targets are no longer supported, though it's unlikely that they were
  still working in the first place.

========================================
14. Performance Improvements
========================================

- A Just-In-Time (JIT) compiler has been added to the opcache extension.
- array_slice() on an array without gaps will no longer scan the whole array to
  find the start offset. This may significantly reduce the runtime of the
  function with large offsets and small lengths.
- strtolower() now uses a SIMD implementation when using the "C" LC_CTYPE
  locale (which is the default).
