## CATransaction

#### Overview

- `CATransaction`is unique in thread.
- Implicit transaction is only uesd when there is no  active transaction while change the layer tree.
- `CATransaction`capture layer tree changes  just  by mark the root layer.

####  Topics

- `+[CATransaction begin]`    just add 1 in specific var.
- Change layer tree. 
- `+[CATransaction commit]` sub 1 in specific var. if var is 0, then commit actally. Ensure nested transaction are commited completely.

#### Tips

Change the layer follows 3 step.

1. CA::Layer::begin_change(CA::Transaction\*,unsigned int,objc_object\*&)

   - call `CA::Transaction::ensure_implicit()` make sure that add an implicit transaction if there is no active transaction.
   - call `-[CALayer actionForkKey:]`,find the CAAnimation correspond with the property. Store it to objc_object*&.

2. Change property.

3. CA::Layer::end_change(CA::Transaction*,unsigned int,,objc_object\*&),

   - call `CA::Transaction::add_root()` mark the root layer of  the updated layer tree

   - call `CA::Layer::add_animation()` set CATransaction values to the animation(like duration,timingFunction.just guess.)

     

   ​       

   

    

​    