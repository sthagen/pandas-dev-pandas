.. _user_defined_functions:

{{ header }}

*****************************
User-Defined Functions (UDFs)
*****************************

In pandas, User-Defined Functions (UDFs) provide a way to extend the library’s
functionality by allowing users to apply custom computations to their data. While
pandas comes with a set of built-in functions for data manipulation, UDFs offer
flexibility when built-in methods are not sufficient. These functions can be
applied at different levels: element-wise, row-wise, column-wise, or group-wise,
and behave differently, depending on the method used.

Here’s a simple example to illustrate a UDF applied to a Series:

.. ipython:: python

    s = pd.Series([1, 2, 3])

    # Simple UDF that adds 1 to a value
    def add_one(x):
        return x + 1

    # Apply the function element-wise using .map
    s.map(add_one)

You can also apply UDFs to an entire DataFrame. For example:

.. ipython:: python

    df = pd.DataFrame({"A": [1, 2, 3], "B": [10, 20, 30]})

    # UDF that takes a row and returns the sum of columns A and B
    def sum_row(row):
        return row["A"] + row["B"]

    # Apply the function row-wise (axis=1 means apply across columns per row)
    df.apply(sum_row, axis=1)


Why Not To Use User-Defined Functions
-------------------------------------

While UDFs provide flexibility, they come with significant drawbacks, primarily
related to performance and behavior. When using UDFs, pandas must perform inference
on the result, and that inference could be incorrect. Furthermore, unlike vectorized operations,
UDFs are slower because pandas can't optimize their computations, leading to
inefficient processing.

.. note::
    In general, most tasks can and should be accomplished using pandas’ built-in methods or vectorized operations.

Despite their drawbacks, UDFs can be helpful when:

* **Custom Computations Are Needed**: Implementing complex logic or domain-specific calculations that pandas'
  built-in methods cannot handle.
* **Extending pandas' Functionality**: Applying external libraries or specialized algorithms unavailable in pandas.
* **Handling Complex Grouped Operations**: Performing operations on grouped data that standard methods do not support.

For example:

.. code-block:: python

    from sklearn.linear_model import LinearRegression

    # Sample data
    df = pd.DataFrame({
        'group': ['A', 'A', 'A', 'B', 'B', 'B'],
        'x': [1, 2, 3, 1, 2, 3],
        'y': [2, 4, 6, 1, 2, 1.5]
    })

    # Function to fit a model to each group
    def fit_model(group):
        model = LinearRegression()
        model.fit(group[['x']], group['y'])
        group['y_pred'] = model.predict(group[['x']])
        return group

    result = df.groupby('group').apply(fit_model)


Methods that support User-Defined Functions
-------------------------------------------

User-Defined Functions can be applied across various pandas methods:

+----------------------------+------------------------+--------------------------+----------------------------------------------------------------------------------------------------------------------------------------------+
| Method                     | Function Input         | Function Output          | Description                                                                                                                                  |
+============================+========================+==========================+==============================================================================================================================================+
| :meth:`map`                | Scalar                 | Scalar                   | Apply a function to each element                                                                                                             |
+----------------------------+------------------------+--------------------------+----------------------------------------------------------------------------------------------------------------------------------------------+
| :meth:`apply` (axis=0)     | Column (Series)        | Column (Series)          | Apply a function to each column                                                                                                              |
+----------------------------+------------------------+--------------------------+----------------------------------------------------------------------------------------------------------------------------------------------+
| :meth:`apply` (axis=1)     | Row (Series)           | Row (Series)             | Apply a function to each row                                                                                                                 |
+----------------------------+------------------------+--------------------------+----------------------------------------------------------------------------------------------------------------------------------------------+
| :meth:`agg`                | Series/DataFrame       | Scalar or Series         | Aggregate and summarizes values, e.g., sum or custom reducer                                                                                 |
+----------------------------+------------------------+--------------------------+----------------------------------------------------------------------------------------------------------------------------------------------+
| :meth:`transform` (axis=0) | Column (Series)        | Column(Series)           | Same as :meth:`apply` with (axis=0), but it raises an exception if the function changes the shape of the data                                |
+----------------------------+------------------------+--------------------------+----------------------------------------------------------------------------------------------------------------------------------------------+
| :meth:`transform` (axis=1) | Row (Series)           | Row (Series)             | Same as :meth:`apply` with (axis=1), but it raises an exception if the function changes the shape of the data                                |
+----------------------------+------------------------+--------------------------+----------------------------------------------------------------------------------------------------------------------------------------------+
| :meth:`filter`             | Series or DataFrame    | Boolean                  | Only accepts UDFs in group by. Function is called for each group, and the group is removed from the result if the function returns ``False`` |
+----------------------------+------------------------+--------------------------+----------------------------------------------------------------------------------------------------------------------------------------------+
| :meth:`pipe`               | Series/DataFrame       | Series/DataFrame         | Chain functions together to apply to Series or Dataframe                                                                                     |
+----------------------------+------------------------+--------------------------+----------------------------------------------------------------------------------------------------------------------------------------------+

When applying UDFs in pandas, it is essential to select the appropriate method based
on your specific task. Each method has its strengths and is designed for different use
cases. Understanding the purpose and behavior of each method will help you make informed
decisions, ensuring more efficient and maintainable code.

.. note::
    Some of these methods are can also be applied to groupby, resample, and various window objects.
    See :ref:`groupby`, :ref:`resample()<timeseries>`, :ref:`rolling()<window>`, :ref:`expanding()<window>`,
    and :ref:`ewm()<window>` for details.


:meth:`DataFrame.apply`
~~~~~~~~~~~~~~~~~~~~~~~

The :meth:`apply` method allows you to apply UDFs along either rows or columns. While flexible,
it is slower than vectorized operations and should be used only when you need operations
that cannot be achieved with built-in pandas functions.

When to use: :meth:`apply` is suitable when no alternative vectorized method or UDF method is available,
but consider optimizing performance with vectorized operations wherever possible.

:meth:`DataFrame.agg`
~~~~~~~~~~~~~~~~~~~~~

If you need to aggregate data, :meth:`agg` is a better choice than apply because it is
specifically designed for aggregation operations.

When to use: Use :meth:`agg` for performing custom aggregations, where the operation returns
a scalar value on each input.

:meth:`DataFrame.transform`
~~~~~~~~~~~~~~~~~~~~~~~~~~~

The :meth:`transform` method is ideal for performing element-wise transformations while preserving the shape of the original DataFrame.
It is generally faster than apply because it can take advantage of pandas' internal optimizations.

When to use: When you need to perform element-wise transformations that retain the original structure of the DataFrame.

.. code-block:: python

    from sklearn.linear_model import LinearRegression

    df = pd.DataFrame({
        'group': ['A', 'A', 'A', 'B', 'B', 'B'],
        'x': [1, 2, 3, 1, 2, 3],
        'y': [2, 4, 6, 1, 2, 1.5]
    }).set_index("x")

    # Function to fit a model to each group
    def fit_model(group):
        x = group.index.to_frame()
        y = group
        model = LinearRegression()
        model.fit(x, y)
        pred = model.predict(x)
        return pred

    result = df.groupby('group').transform(fit_model)

:meth:`DataFrame.filter`
~~~~~~~~~~~~~~~~~~~~~~~~

The :meth:`filter` method is used to select subsets of the DataFrame’s
columns or row. It is useful when you want to extract specific columns or rows that
match particular conditions.

When to use: Use :meth:`filter` when you want to use a UDF to create a subset of a DataFrame or Series

.. note::
    :meth:`DataFrame.filter` does not accept UDFs, but can accept
    list comprehensions that have UDFs applied to them.

.. ipython:: python

    # Sample DataFrame
    df = pd.DataFrame({
        'AA': [1, 2, 3],
        'BB': [4, 5, 6],
        'C': [7, 8, 9],
        'D': [10, 11, 12]
    })

    # Function that filters out columns where the name is longer than 1 character
    def is_long_name(column_name):
        return len(column_name) > 1

    df_filtered = df.filter(items=[col for col in df.columns if is_long_name(col)])
    print(df_filtered)

Since filter does not directly accept a UDF, you have to apply the UDF indirectly,
for example, by using list comprehensions.

:meth:`DataFrame.map`
~~~~~~~~~~~~~~~~~~~~~

The :meth:`map` method is used specifically to apply element-wise UDFs.

When to use: Use :meth:`map` for applying element-wise UDFs to DataFrames or Series.

:meth:`DataFrame.pipe`
~~~~~~~~~~~~~~~~~~~~~~

The :meth:`pipe` method is useful for chaining operations together into a clean and readable pipeline.
It is a helpful tool for organizing complex data processing workflows.

When to use: Use :meth:`pipe` when you need to create a pipeline of operations and want to keep the code readable and maintainable.


Performance
-----------

While UDFs provide flexibility, their use is generally discouraged as they can introduce
performance issues, especially when written in pure Python. To improve efficiency,
consider using built-in ``NumPy`` or ``pandas`` functions instead of UDFs
for common operations.

.. note::
    If performance is critical, explore **vectorized operations** before resorting
    to UDFs.

Vectorized Operations
~~~~~~~~~~~~~~~~~~~~~

Below is a comparison of using UDFs versus using Vectorized Operations:

.. code-block:: python

    # User-defined function
    def calc_ratio(row):
        return 100 * (row["one"] / row["two"])

    df["new_col"] = df.apply(calc_ratio, axis=1)

    # Vectorized Operation
    df["new_col2"] = 100 * (df["one"] / df["two"])

Measuring how long each operation takes:

.. code-block:: text

    User-defined function:  5.6435 secs
    Vectorized:             0.0043 secs

Vectorized operations in pandas are significantly faster than using :meth:`DataFrame.apply`
with UDFs because they leverage highly optimized C functions
via ``NumPy`` to process entire arrays at once. This approach avoids the overhead of looping
through rows in Python and making separate function calls for each row, which is slow and
inefficient. Additionally, ``NumPy`` arrays benefit from memory efficiency and CPU-level
optimizations, making vectorized operations the preferred choice whenever possible.


Improving Performance with UDFs
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

In scenarios where UDFs are necessary, there are still ways to mitigate their performance drawbacks.
One approach is to use **Numba**, a Just-In-Time (JIT) compiler that can significantly speed up numerical
Python code by compiling Python functions to optimized machine code at runtime.

By annotating your UDFs with ``@numba.jit``, you can achieve performance closer to vectorized operations,
especially for computationally heavy tasks.

.. note::
    You may also refer to the user guide on `Enhancing performance <https://pandas.pydata.org/pandas-docs/dev/user_guide/enhancingperf.html#numba-jit-compilation>`_
    for a more detailed guide to using **Numba**.

Using :meth:`DataFrame.pipe` for Composable Logic
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Another useful pattern for improving readability and composability, especially when mixing
vectorized logic with UDFs, is to use the :meth:`DataFrame.pipe` method.

:meth:`DataFrame.pipe` doesn't improve performance directly, but it enables cleaner
method chaining by passing the entire object into a function. This is especially helpful
when chaining custom transformations:

.. code-block:: python

    def add_ratio_column(df):
        df["ratio"] = 100 * (df["one"] / df["two"])
        return df

    df = (
        df
        .query("one > 0")
        .pipe(add_ratio_column)
        .dropna()
    )

This is functionally equivalent to calling ``add_ratio_column(df)``, but keeps your code
clean and composable. The function you pass to :meth:`DataFrame.pipe` can use vectorized operations,
row-wise UDFs, or any other logic; :meth:`DataFrame.pipe` is agnostic.

.. note::
    While :meth:`DataFrame.pipe` does not improve performance on its own,
    it promotes clean, modular design and allows both vectorized and UDF-based logic
    to be composed in method chains.
