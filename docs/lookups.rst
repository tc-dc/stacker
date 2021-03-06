=======
Lookups
=======

Stacker provides the ability to dynamically replace values in the config via a
concept called lookups. A lookup is meant to take a value and convert
it by calling out to another service or system.

A lookup is denoted in the config with the ``${<lookup type> <lookup
input>}`` syntax. If ``<lookup type>`` isn't provided, the default of
``output`` will be used.

Lookups are only resolved within `Variables
<terminology.html#variables>`_. They can be nested in any part of a YAML
data structure and within another lookup itself.

.. note::
  If a lookup has a non-string return value, it can be the only lookup
  within a value.

  ie. if `custom` returns a list, this would raise an exception::

    Variable: ${custom something}, ${output otherStack::Output}

  This is valid::

    Variable: ${custom something}


For example, given the following::

  stacks:
    - name: sg
      class_path: some.stack.blueprint.Blueprint
      variables:
        Roles:
          - ${output otherStack::IAMRole}
        Values:
          Env:
            Custom: ${custom ${output otherStack::Output}}
            DBUrl: postgres://${output dbStack::User}@${output dbStack::HostName}

The Blueprint would have access to the following resolved variables
dictionary::

  # variables
  {
    "Roles": ["other-stack-iam-role"],
    "Values": {
      "Env": {
        "Custom": "custom-output",
        "DBUrl": "postgres://user@hostname",
      },
    },
  }

stacker includes the following lookup types:

  - output_
  - kms_
  - xref_
  - rxref

.. _output:

Output Lookup
-------------

The ``output`` lookup takes a value of the format:
``<stack name>::<output name>`` and retrieves the output from the given stack
name within the current namespace.

stacker treats output lookups differently than other lookups by auto
adding the referenced stack in the lookup as a requirement to the stack
whose variable the output value is being passed to.

You can specify an output lookup with the following syntax::

  ConfVariable: ${output someStack::SomeOutput}

.. _kms:

KMS Lookup
----------

The ``kms`` lookup type decrypts its input value.

As an example, if you have a database and it has a parameter called
``DBPassword`` that you don't want to store in clear text in your config
(maybe because you want to check it into your version control system to
share with the team), you could instead encrypt the value using ``kms``.

For example::

  # We use the aws cli to get the encrypted value for the string
  # "PASSWORD" using the master key called 'myStackerKey' in us-east-1
  $ aws --region us-east-1 kms encrypt --key-id alias/myStackerKey \
      --plaintext "PASSWORD" --output text --query CiphertextBlob

  CiD6bC8t2Y<...encrypted blob...>

  # In stacker we would reference the encrypted value like:
  DBPassword: ${kms us-east-1@CiD6bC8t2Y<...encrypted blob...>}

  # The above would resolve to
  DBPassword: PASSWORD

This requires that the person using stacker has access to the master key used
to encrypt the value.

It is also possible to store the encrypted blob in a file (useful if the
value is large) using the ``file://`` prefix, ie::

  DockerConfig: ${kms file://dockercfg}

.. note::
  Lookups resolve the path specified with `file://` relative to
  the location of the config file, not where the stacker command is run.

.. _xref:

XRef Lookup
-----------

The ``xref`` lookup type is very similar to the ``output`` lookup type, the
difference being that ``xref`` resolves output values from stacks that
aren't contained within the current namespace.

The ``output`` type will take a stack name and use the current context to
expand the fully qualified stack name based on the namespace. ``xref``
skips this expansion because it assumes you've provided it with
the fully qualified stack name already. This allows you to reference
output values from any CloudFormation stack.

Also, unlike the ``output`` lookup type, ``xref`` doesn't impact stack
requirements.

For example::

  ConfVariable: ${xref fully-qualified-stack::SomeOutput}

.. file:

.. _rxref:

RXRef Lookup
-----------

The ``rxref`` lookup type is very similar to the ``output`` and ``xref`` lookup
type, the difference being that ``rxref`` resolves output values from stacks
that are relative to the current namespace but external to the stack.

The ``output`` type will take a stack name prefixed by the namespace
and use the current context to expand the fully qualified stack name
based on the namespace. ``rxref`` skips this expansion because it assumes
you've provided it with the fully qualified stack name already. This allows
you to reference output values from any CloudFormation stack.

Also, unlike the ``output`` lookup type, ``rxref`` doesn't impact stack
requirements.

For example::

  ConfVariable: ${rxref fully-qualified-stack::SomeOutput}

.. file:

File Lookup
-----------

The ``file`` lookup type allows the loading of arbitrary data from files on
disk. The lookup additionally supports using a ``codec`` to manipulate or
wrap the file contents prior to injecting it. The parameterized-b64 ``codec``
is particularly useful to allow the interpolation of CloudFormation parameters
in a UserData attribute of an instance or launch configuration.

Basic examples::

  # We've written a file to /some/path:
  $ echo "hello there" > /some/path

  # In stacker we would reference the contents of this file with the following
  conf_key: ${file plain:file://some/path}

  # The above would resolve to
  conf_key: hello there

  # Or, if we used wanted a base64 encoded copy of the file data
  conf_key: ${file base64:file://some/path}

  # The above would resolve to
  conf_key: aGVsbG8gdGhlcmUK

Supported codecs:
 - plain
 - base64 - encode the plain text file at the given path with base64 prior
   to returning it
 - parameterized - the same as plain, but additionally supports
   referencing CloudFormation parameters to create userdata that's
   supplemented with information from the template, as is commonly needed
   in EC2 UserData. For example, given a template parameter of BucketName,
   the file could contain the following text::

     #!/bin/sh
     aws s3 sync s3://{{BucketName}}/somepath /somepath

   and then you could use something like this in the YAML config file::

     UserData: ${file parameterized:/path/to/file}

   resulting in the UserData parameter being defined as::

     { "Fn::Join" : ["", [
       "#!/bin/sh\naws s3 sync s3://",
       {"Ref" : "BucketName"},
       "/somepath /somepath"
     ]] }

 - parameterized-b64 - the same as parameterized, with the results additionally
   wrapped in { "Fn::Base64": ... } , which is what you actually need for
   EC2 UserData

When using parameterized-b64 for UserData, you should use a local_parameter defined
as such::

  from troposphere import AWSHelperFn

  "UserData": {
    "type": AWSHelperFn,
    "description": "Instance user data",
    "default": Ref("AWS::NoValue")
  }

and then assign UserData in a LaunchConfiguration or Instance to self.get_variables()["UserData"].
Note that we use AWSHelperFn as the type because the parameterized-b64 codec returns either a
Base64 or a GenericHelperFn troposphere object.

Custom Lookups
--------------

Custom lookups can be registered within the config. For more information
see `Configuring Lookups <config.html#lookups>`_.
