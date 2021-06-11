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
