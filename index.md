## Week 3 - Block and FlowGraph level eval 

Last week I disabled parameter level evaluation for untrusted flowgraphs, using the evaluated values I've stored in the .grc-files instead. This week I've extended this further by also disabling evaluation on block level as well as flowgraph level.

For the block level evaluation there are two parts, expressions in the block templates as well as import-blocks. The GRC core uses `exec` to run the imports which updates the namespace. Though, this could be exploited to have the GRC run some malicious code and thus need to be disabled for the View-Only Mode. Similarly the templates contain expressions that also could contain malicious code.
See line **25** in the following template (`grc/blocks/pad_source.block.yml`) for an example of a (benign) expression:

```
 1 id: pad_source
 2 label: Pad Source
 3 flags: [ python, cpp ]
 4
 5 parameters:
 6 -   id: label
 7     label: Label
20     ...
21 -   id: vlen
22     label: Vector Length
23     dtype: int
24     default: '1'
25     hide: ${ 'part' if vlen == 1 else 'none' }
26 -   id: num_streams
27     ...
```

These expressions are evaluated using the block's `evaluate` method which before my changes just called the parent flowgraph's evaluate with the block's namespace. I've changed the method to, in a trusted flowgraph, save all expression-evaluation pairs in a dictionary `_eval_cache` which is then used when exporting the block data. For an untrusted flowgraph the block fetches the evaluated expressions from the file with the `import_data` method and `evaluate` then simply returns the stored value.

```Python
def evaluate(self, expr):
    if self.parent_flowgraph.is_trusted:
        self._eval_cache[expr] = self.parent_flowgraph.evaluate(expr, self.namespace)

    evaluated = self._eval_cache.get(expr)

    if evaluated is None:
        # error

    return evaluated

def export_data(self):
    """
    Export this block's params to nested data.

    Returns:
        a nested data odict
    """
    data = collections.OrderedDict()

    # ...

    data['expressions'] = collections.OrderedDict(sorted(
        (expr, val) for expr, val in self._eval_cache.items()
    ))

    return data
```

As for the import blocks the `exec` is only executed if the flowgraph isn't trusted, otherwise it is ignored. For the imports this is sufficient as the View-Only Mode removes the other evaling where the updated namespace provided by the import would have been needed.

```Python
def rewrite(self):
    # ...
    
    if self.parent_flowgraph.is_trusted:
        exec(imports, self.block_namespace)

    # ...
```

Similarly the FlowGraph's rewrite method (`grc/core/FlowGraph.py`) is altered to not perform `renew_namespace` unless trusted. This stops the execs and evals done on the flowgraph level (such as imports):

```Python
def rewrite(self):
    """
    Flag the namespace to be renewed.
    """
    if self.is_trusted:
        self.renew_namespace()
    Element.rewrite(self)

def renew_namespace(self):
    namespace = {}
    # Load imports
    for expr in self.imports():
        try:
            exec(expr, namespace)
        except ImportError:
            # We do not have a good way right now to determine if an import is for a
            # hier block, these imports will fail as they are not in the search path
            # this is ok behavior, unfortunately we could be hiding other import bugs
            pass
        except Exception:
            log.exception('Failed to evaluate import expression "{0}"'.format(expr), exc_info=True)
            pass

    self.imported_names = list(namespace.keys())

    # ...
```

The next week's work will focus on refining this further, adding a trigger for when the trust-level should change from untrusted to trusted (later in form of a GUI prompt), as well as writing a test using the Python audit features to make sure evals/execs are disabled as they should.

\- Oscar

## Week 2 - Using the saved values instead of evaluating
This week I have improved the exporting of evaluated values that I implemented last week as well as started working on actually using these values when loading the grc-file.

### Exporting non-primitives
As mentioned last week PyYAML can dump primitive types directly without problems wheras non-primitives such as [Constellations](https://wiki.gnuradio.org/index.php/Constellation_Object) needs to be represented as a string before being dumped. This is implemented using a type whitelist which determines how a parameter is represented depending on its type. If it is primitive it is
represented with it's evaluated value wheras non-primitives are represented using the parameter's pretty\_print-method (moved from `grc/gui/canvas/param.py` to `grc/core/param.py)

```Python
def export_data(self):
    data = collections.OrderedDict()

    # ...

    # whitelist of types accepted by YAML
    yaml_types = PARAM_TYPE_NAMES - {'raw'}

    def get_param_repr(param):
        if param.dtype in yaml_types:
            return param.get_evaluated()
        else:
            return param.pretty_print()

    data['evaluated'] = collections.OrderedDict(sorted(
        (param_id, get_param_repr(param)) for param_id, param in self.params.items()
        if (param_id != 'id' or self.key == 'options')
    ))

    return data
```

### Using the evaluated values
As mentioned I've started limiting the parameter evaluation, using the stored value instead. This is done on a parameter level by adding a field containing the saved value in the `Param` class and initializing it in the block's `import\_data` method. The param's evaluate-method is modified to use the saved value if the flowgraph isn't trusted, and continue with the evaluation if it is. The saved value is stored in a separate field than `\_evaluated` as `\_evaluated` is reset during rewrites of the graph/params.

Currently the trust setting is hardcoded in the flowgraph but a more extensive and GUI-accessible trust system will be implemented later in the project.

```
def evaluate(self):
    """
    Evaluate the value.

    Returns:
        evaluated type
    """
    self._init = True
    self._lisitify_flag = False
    self._stringify_flag = False
    dtype = self.dtype
    expr = self.get_value()
    scale_factor = self.scale_factor

    if not self.parent_flowgraph.is_trusted:
        self._evaluated = self._saved_evaluated
        return self.get_saved_evaluated()

    # some evaluation...
```

This stops parameter evaluation which is a good start but not complete. In addition to this there are import-blocks which are executed as well as expressions in the block templates which are evaluated. The next week's work will be to disable these as well.

### About the View-Only Mode
Looking ahead in the project, when the core-work is done, the project will shift into GUI and UX work. The main features of the UX are:
- a trust-dialog asking if the user trusts the flowgraph
- an error dialog for whenever something goes wrong, specifically when the stored values are not enough to display the graph without some evaluation

and if time permits:
- a dialog showing the expressions that will be run if the user chooses to trust the file. This could possibly be enhanced with heuristics trying to guess which expressions may be of special interest and highlight these.

For more details you can read my [project proposal](https://docs.google.com/document/d/1dL6PziJSopcY3O7gJ6CXiedTSdbhrHVFhR-UJRTmsng).

That is it for now. Thanks for reading and please reach out if you have any questions!

\- Oscar

PS: Interestingly enoguh VS Code recently launched a [feature which allows a user to trust a folder/files to enable features that may automatically execute code in the folder/file](https://code.visualstudio.com/updates/v1_57#_workspace-trust)

## Week 1 - Exporting evaluated values
To start off I want to thank everyone in the community for the warm welcome. It is reassuring to hear how much support there is!

I will be posting an update on here every friday of GSoC and any code I write can be viewed in my fork:
https://github.com/oscekh/gnuradio/tree/feature/grc-view-only-mode

For the first week of coding I am working on saving the evaluated block parameters in the .grc-files alongside the previously saved data, such as parameter expressions. The purpose of this is to make the GRC display an opened flowgraph using the saved values initially instead of evaluating possibly harmful Python expressions.

The resulting piece of code is very simple so far, currently the [block's export_data method](https://github.com/oscekh/gnuradio/blob/feature/grc-view-only-mode/grc/core/blocks/block.py#L634) is modified to fetch the parameter objects' evaluated values and pass them along in the exported data dictionary. See:

```Python
    def export_data(self):
        """
        Export this block's params to nested data.
        Returns:
            a nested data odict
        """
        data = collections.OrderedDict()
        if self.key != 'options':
            data['name'] = self.name
            data['id'] = self.key

        data['parameters'] = collections.OrderedDict(sorted(
            (param_id, param.value) for param_id, param in self.params.items()
            if (param_id != 'id' or self.key == 'options')
        ))

        data['states'] = collections.OrderedDict(sorted(self.states.items()))

        data['evaluated'] = collections.OrderedDict(sorted(
            (param_id, param.get_evaluated()) for param_id, param in self.params.items()
            if (param_id != 'id' or self.key == 'options')
        ))

        return data
```

This code relies on PyYAML knowing how to deal with whatever value is returned by `param.get_evaluated()` which results in little code working well for primitive types such as ints, strings, and lists, but likely not as well for custom types such as [Constellations](https://wiki.gnuradio.org/index.php/Constellation_Object). Thus, the exportation will need further work and testing for such
cases. Currently the plan is to implement some sort of type-whitelist which determines how types are stored (i.e. using yamls representation or a string representation etc).

This continuation will also go hand in hand with the next step, which is reading the values and using them instead of evaluating the expressions. That will also get somewhat tricky due to how the different types are handled differently and as it is not obvious how to prevent the evaluation elegantly.

To aid my debugging and tracing of the evaluation control flow I am trying out the [Python 3.8 Audit feature](https://www.python.org/dev/peps/pep-0578/) with an audit hook on the `exec` event, as well as using general logging. Though, the auditing is only for development purposes, otherwise it would require upping the minimum python version supported by GNU Radio.

```Python
def audit(event, args):
    if event == 'exec':
        # args is the code object
        log.debug("evaluating", args)
```


For the next few weeks the work will be more intricate and I will have more interesting technical details to share in my updates here. Feel free to reach out if you have any questions or feedback!

\- Oscar



## GSoC'21 Blog - Implementing a View-Only Mode in the GnuRadio Companion (GRC)

Welcome to my blog, here I will provide weekly progress updates on my work on the GRC for Google Summer of Code.

Coding will begin on June 7th, the period before that is for community bonding and getting comfortable with the codebase to get ready for coding. This includes studying the code base, solving smaller issues and discussing with the mentors.

\- Oscar Ekholm
