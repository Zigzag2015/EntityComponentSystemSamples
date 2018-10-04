# How to optimize for the Burst compiler

* Use Unity.Mathematics, Burst natively understands the math operations and is optimized for it.
* Avoid branches. Use math.min, math.max, math.select instead.
* For jobs that have to be highly optimized, ensure that each job uses every single variable in the IComponentData. If some variables in an IComponentData is not being used, move it to a separate component. That way the unused data will not be loaded into cache lines when iterating over Entities.



*使用Unity.Mathematics，Burst本身可以理解数学运算并针对它进行了优化。
*避免分支机构。 请改用math.min，math.max，math.select。
*对于必须高度优化的作业，请确保每个作业使用IComponentData中的每个变量。 如果未使用IComponentData中的某些变量，请将其移至单独的组件。 这样，在迭代实体时，未使用的数据将不会加载到缓存行中。