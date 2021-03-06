
:mod:`disco.core` --- Client interface for Disco
================================================

.. module:: disco.core
   :synopsis: Client interface for Disco

The :mod:`disco.core` module provides a high-level interface for
communication with the Disco master. It provides functions for submitting
new jobs, querying status of the system, and getting results of jobs.

The :class:`Disco` object encapsulates connection to the Disco
master. Once a connection has been established, you can use the
object to query status of the system, or submit a new job with the
:meth:`Disco.new_job` method. See the :mod:`disco.func` module for more
information about constructing Disco jobs.

:meth:`Disco.new_job` is provided with all information needed to run
a job, which it packages and sends to the master. The method returns
immediately and returns a :class:`Job` object that corresponds to the
newly started job.

All methods in :class:`Disco` that are related to individual jobs, namely

 - :meth:`Disco.wait`
 - :meth:`Disco.jobinfo`
 - :meth:`Disco.results`
 - :meth:`Disco.jobspec`
 - :meth:`Disco.clean`
 - :meth:`Disco.kill`

are also accessible through the :class:`Job` object, so you can say
`job.wait()` instead of `disco.wait(job.name)`. However, the job methods
in :class:`Disco` come in handy if you want to manipulate a job that is
identified by a job name (:attr:`Job.name`) instead of a :class:`Job`
object.

If you have access only to results of a job, you can extract the job
name from an address with the :func:`disco.util.jobname` function. A typical
case is that you are done with results of a job and they are not needed
anymore. You can delete the unneeded job files as follows::
        
        from disco.core import Disco
        from disco.util import jobname

        Disco(master).purge(jobname(results[0]))


:class:`Disco` --- Interface to the Disco master
------------------------------------------------

.. class:: Disco(host)

   Opens and encapsulates connection to the Disco master.

   *host* is the address of the Disco master, for instance
   ``disco://localhost``. See :func:`disco.util.disco_host` for more
   information on how *host* is interpreted.

   .. method:: Disco.request(url[, data, raw_handle])

   Requests *url* at the master. If a string *data* is specified, a POST request
   is made with *data* as the request payload. If *raw_handle* is set to *True*,
   a file handle to the results is returned. By default a string is returned
   that contains the reply for the request. This method is mostly used by other
   methods in this class internally.

   .. method:: Disco.nodeinfo()

   Returns a dictionary describing status of the nodes that are managed by
   this Disco master.
   
   .. method:: Disco.joblist()

   Returns a list of jobs and their statuses.

   .. method:: Disco.kill(name)

   Kills the job *name*.

   .. method:: Disco.clean(name)

   Cleans records of the job *name*. Note that after the job records
   have been cleaned, there is no way to obtain addresses to the result
   files from the master. However, no files are actually deleted by
   :meth:`Disco.clean`, in contrast to :meth:`Disco.purge`. This function
   provides a way to notify the master not to bother about the job anymore,
   although you want to keep results of the job for future use. If you
   won't need the results, use :meth:`Disco.purge`.

   .. method:: Disco.purge(name)

   Deletes all records and files related to the job *name*. This implies
   :meth:`Disco.clean`.

   .. method:: Disco.jobspec(name)

   Returns the raw job request package, as constructed by
   :meth:`Disco.new_job`, for the job *name*.

   .. method:: Disco.results(name)

   Returns the list of result files for the job *name*, if available.

   .. method:: Disco.jobinfo(name)

   Returns a dictionary containing information about the job *name*.

   .. method:: Disco.wait(name[, poll_interval, timeout, clean])

   Block until the job *name* has finished. Returns a list URLs to the
   results files which is typically processed with :func:`result_iterator`.
   
   :meth:`Disco.wait` polls the server for the job status every
   *poll_interval* seconds. It raises a :class:`disco.JobException` if the
   job hasn't finished in *timeout* seconds, if specified.
   
   *clean* is a convenience parameter which, if set to `True`,
   calls :meth:`Disco.clean` when the job has finished. This makes
   it possible to execute a typical Disco job in one line::
   
        results = disco.new_job(...).wait(clean = True)

   Note that this only removes records from the master, but not the
   actual result files. Once you are done with the results, call::

        disco.purge(disco.util.jobname(results[0]))

   to delete the actual result files.

   .. method:: Disco.new_job(...)

   Submits a new job request to the master. This method accepts the same
   set of keyword as the constructor of the :class:`Job` object below. The
   `master` argument for the :class:`Job` constructor is provided by
   this method. Returns a :class:`Job` object that corresponds to the
   newly submitted job request.

:class:`Job` --- Disco job
--------------------------

.. class:: Job(master, [name, input_files, fun_map, map_reader, reduce, partition, combiner, nr_maps, nr_reduces, sort, params, mem_sort_limit, async, clean, chunked, ext_params, required_modules, status_interval])

   Starts a new Disco job. You seldom instantiate this class
   directly. Instead, the :meth:`Disco.new_job` is used to start a job
   on a particular Disco master. :meth:`Disco.new_job` accepts the same
   set of keyword arguments as specified below.

   The constructor returns immediately after a job request has been
   submitted. A typical pattern in Disco scripts is to run a job
   synchronously, that is, to block the script until the job has
   finished. This is accomplished as follows::
        
        from disco.core import Disco
        results = Disco(master).new_job(...).wait(clean = True)

   Note that job methods of the :class:`Disco` class are directly
   accessible through the :class:`Job` object, such as :meth:`Disco.wait`
   above.

   The constructor raises a :class:`JobException` if an error occurs
   when the job is started.

   All arguments that are required are marked as such. All other arguments
   are optional.

     * *master* - an instance of the :class:`Disco` class that identifies
       the Disco master runs this job. This argument is required but
       it is provided automatically when the job is started using
       :meth:`Disco.new_job`.

     * *name* - the job name (**required**). The ``@[timestamp]`` suffix is appended
       to the name to ensure uniqueness. If you start more than one job
       per second, you cannot rely on the timestamp which increments only
       once per second. In any case, users are strongly recommended to devise a
       good naming scheme of their own. Only characters in ``[a-zA-Z0-9_]``
       are allowed in the job name.

     * *input_files* - a list of input files for the map function (**required**). Each
       input must be specified in one of the following four protocols:

         * ``http://www.example.com/data`` - any HTTP address
         * ``disco://cnode03/bigtxt/file_name`` - Disco address. Refers to ``cnode03:/var/disco/bigtxt/file_name``. Currently this is an alias for ``http://cnode03:8989/bigtxt/file_name``.
         * ``dir://cnode03/jobname/`` - Result directory. This format is used by Disco internally.
         * ``/home/bob/bigfile.txt`` - a local file. Note that the file must either exist on all the nodes or you must make sure that the job is run only on the nodes where the file exists. Due to these restrictions, this form has only limited use.

     * *fun_map* - a :term:`pure function` that defines the map task (**required**). 
       The function takes two parameters, an input entry and a parameter object,
       and it outputs a list of key-value pairs in tuples. For instance::

                def fun_map(e, params):
                        return [(w, 1) for w in e.split()]

       This example takes a line of text as input in *e*, tokenizes it, and returns
       a list of words as the output. The argument *params* is the object
       specified by *params* in :func:`disco.job`. It may be used to maintain state
       between several calls to the map function.

       The map task can also be an external program. For more information, see
       :ref:`discoext`.
        
     * *map_reader* - a function that parses input entries from an input file. By
       default :func:`disco.map_line_reader`. The function is defined as follows::

                def map_reader(fd, size, fname)

       where *fd* is a file object connected to the input file, *size* is the input
       size (may be *None*), and *fname* is the input file name. The reader function
       must read at most *size* bytes from *fd*. The function parses the stream and
       yields input entries to the map function.

       Disco worker provides a convenience function :func:`disco.func.re_reader`
       that can be used to create parser based on regular expressions.

       If you want to use outputs of an earlier job as inputs, use
       :func:`disco.func.chain_reader` as the *map_reader*.

     * *reduce* - a :term:`pure function` that defines the reduce task. The
       function takes three parameters, an iterator to the intermediate
       key-value pairs produced by the map function. an output object that
       handles the results, and a parameter object. For instance::

                def fun_reduce(iter, out, params):
                        d = {}
                        for w, c in iter:
                                if w in d:
                                        d[w] += 1
                                else:
                                        d[w] = 1
                        for w, c in d.iteritems():
                                out.add(w, c)
      
       Counts how many teams each key appears in the intermediate results.

       By default no reduce function is specified and the job will quit after
       the map functions have finished.
       
       The reduce task can also be an external program. For more
       information, see :ref:`discoext`.

     * *partition* - a :term:`pure function` that defines the partitioning
       function, that is, the function that decides how the map outputs
       are distributed to the reduce functions. The function is defined as
       follows::

                def partition(key, nr_reduces, params)

       where *key* is a key returned by the map function and *nr_reduces* the
       number of reduce functions. The function returns an integer between 0 and
       *nr_reduces* that defines to which reduce instance this key-value pair is
       assigned. *params* is an user-defined object as defined by the *params*
       parameter in :meth:`Disco.job`.

       The default partitioning function is :func:`disco.func.default_partition`.

     * *combiner* - a :term:`pure function` that can be used to post-process
       results of the map function. The function is defined as follows::

                def combiner(key, value, comb_buffer, done, params)

       where the first two parameters correspond to a single key-value
       pair from the map function. The third parameter, *comb_buffer*,
       is an accumulator object, a dictionary, that combiner can use to
       save its state. Combiner must control the *comb_buffer* size,
       to prevent it from consuming too much memory, for instance, by
       calling *comb_buffer.clear()* after a block of results has been
       processed. *params* is an user-defined object as defined by the
       *params* parameter in :func:`disco.job`.
       
       Combiner function may return an iterator of key-value pairs
       (tuples) or *None*.

       Combiner function is called after the partitioning function, so
       there are *nr_reduces* separate *comb_buffers*, one for each reduce
       partition. Combiner receives all key-value pairs from the map
       functions before they are saved to intermediate results. Only the
       pairs that are returned by the combiner are saved to the results.

       After the map functions have consumed all input entries,
       combiner is called for the last time with the *done* flag set to
       *True*. This is the last opportunity for the combiner to return
       an iterator to the key-value pairs it wants to output.

     * *nr_maps* - the number of parallel map operations. By default,
       ``nr_maps = len(input_files)``. Note that making this value
       larger than ``len(input_files)`` has no effect. You can only save
       resources by making the value smaller than that.

     * *nr_reduces* - the number of parallel reduce operations. This equals
       to the number of partitions. By default, ``nr_reduces = max(nr_maps / 2, 1)``.

     * *sort* - a boolean value that specifies whether the intermediate results,
       that is, input to the reduce function, should be sorted. Sorting is most
       useful in ensuring that the equal keys are consequent in the input for
       the reduce function.

       Other than ensuring that equal keys are grouped together, sorting
       ensures that numerical keys are returned in the ascending order. No
       other assumptions should be made on the comparison function.

       Sorting is performed in memory, if the total size of the input data
       is less than *mem_sort_limit* bytes. If it is larger, the external
       program ``sort`` is used to sort the input on disk.
       
       False by default.

     * *params* - an arbitrary object that is passed to the map and reduce
       function as the second argument. The object is serialized using the
       *pickle* module, so it should be pickleable.

       A convience class :class:`disco.Params` is provided that
       provides an easy way to encapsulate a set of parameters for the
       functions. As a special feature, :class:`disco.Params` allows
       including functions in the parameters by making them pickleable.

       By default, *params* is an empty :class:`disco.Params` object.

     * *mem_sort_limit* - sets the maximum size for the input that can be sorted
       in memory. The larger inputs are sorted on disk. By default 256MB.

     * *clean* - clean the job records from the master after the results have
       been returned, if the job was succesful. By default true. If set to
       false, you must use either :func:`discoapi.Disco.clean` or the web interface
       manually to clean the job records.

     * *chunked* - if the reduce function is specified, the worker saves
       results from a single map instance to a single file that includes
       key-value pairs for all partitions. When the reduce function is
       executed, the worker knows how to retrieve pairs for each partition
       from the files separately. This is called the chunked mode.

       If no reduce is specified, results for each partition are saved
       to a separate file. This produces *M \* P* files where *M* is the number
       of maps and *P* is the number of reduces. This number can potentially be
       large, so the *chunked* parameter can be used to enable or disable the
       chunked mode, overriding the default behavior.

       Usually there is no need to use this parameter.
     
     * *ext_params* - if either map or reduce function is an external program,
       typically specified using the :func:`disco.external` function, this
       parameter is used to deliver a parameter set to the program.

       The default C interface for external Disco functions uses
       the *netstring* module to encode the parameter set. Hence the
       *ext_params* value must be a dictionary consisting of string-string
       pairs.

       However, if the external program doesn't use the default C
       interface, it can receive parameters in any format. In this case,
       the *ext_params* value can be an arbitrary string which can be
       decoded by the program properly.
       
       For more information, see :ref:`discoext`.

     * *required_modules* - is a list of additional modules (module names) which
       are required by job functions. Modules listed here are imported to the
       functions' namespace.

     * *status_interval* - print out "K items mapped / reduced" for
       every Nth item. By default 100000. Setting the value to 0 disables
       messages.

       Increase this value, or set it to zero, if you get "Message rate limit
       exceeded" error due to system messages. This might happen if your map /
       reduce task is really fast. Decrease the value if you want to follow 
       your task in more real-time or you don't have many data items.

    .. attribute:: Job.name

       Name of the job. You can store or transfer the name string if
       you need to identify the job in another process. In this case,
       you can use the job methods in :class:`Disco` directly.

    .. attribute:: Job.master

       An instance of the :class:`Disco` class that identifies the Disco
       master that runs this job.


.. class:: Params([key = value])

   Parameter container for map / reduce tasks. This object provides a convenient
   way to contain custom parameters, or state, in your tasks. 

   This example shows a simple way of using :class:`Params`::
        
        def fun_map(e, params):
                params.c += 1
                if not params.c % 10:
                        return [(params.f(e), params.c)]
                else:
                        return [(e, params.c)]

        disco.job("disco://localhost:5000",
                  ["disco://localhost/myjob/file1"],
                  fun_map,
                  params = disco.Params(c = 0, f = lambda x: x + "!"))

   You can specify any number of key-value pairs to the :class:`Params`
   constructor.  The pairs will be delivered as-is to map and reduce
   functions through the *params* argument. *Key* must be a valid Python
   identifier but *value* can be any Python object. For instance, *value*
   can be an arbitrary :term:`pure function`, such as *params.f* in the
   previous example.

.. function:: result_iterator(results[, notifier])

   Iterates the key-value pairs in job results. *results* is a list of
   results, as returned by :meth:`Disco.wait`.

   *notifier* is a function that accepts a single parameter, a URL of
   the result file, that is called when the iterator moves to the next
   result file.


.. class:: JobException

   Raised when job fails on Disco master.

   .. attribute:: msg
 
   Error message.

   .. attribute:: JobException.name

   Name of the failed job.

   .. attribute:: JobException.master
   
   Address of the Disco master that produced the error.

