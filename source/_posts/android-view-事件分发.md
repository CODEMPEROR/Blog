---
title: androdid-view-事件分发
date: 2016-08-17 09:52:57
categories:android
tags:["Android", "View事件分发"]
---

# view事件分发
>最重要的四个事件：DOWN, MOVE, UP, CANCEL;


>事件从上往下依次传递：Activity、ViewGroup、View。但Activity和ViewGroup在没有把事件传递给View之前只决定***是否拦截该事件***（即是否将事件传递给view，让view去处理该事件）。如果View不处理此事件，则将该事件传递给父级ViewGroup，依次类推。如果View处理了此事件（例如设置了onClickListener、onTouchListener），则该事件则被View处理，且该事件的后续事件也被View来处理（所谓后续事件是指：用户点击屏幕之后会产生Action_DOWN事件，但如果用户在屏幕上滑动则会产生MOVE事件，离开屏幕则会产生UP事件，取消则会产生CANCEL事件）。
  
>在之前需要明白一个原理：如果某个View不处理DOWN事件，那么系统则不会将该DOWN事件的后续事件分发给该View，这是因为DOWN点击事件是所有屏幕触摸事件的开始，如果该View不处理DOWN事件，那么系统认为该View不想处理该屏幕触摸事件，所以也就不会讲后续事件分发给该View。


>前面提到**是否拦截事件**是由ViewGroup的onInterceptTouchEvent来决定的，该方法仅存在于ViewGroup中。

```
public boolean onInterceptTouchEvent(MotionEvent ev){
	return false;
}
```


>返回**false**则表示不拦截屏幕触摸事件，触摸事件可以按照正常流程分发给View，否则该ViewGroup则会拦截该事件。所以如果想让某个ViewGroup拦截所有的触摸事件，则继承该ViewGroup使onInterceptTouchEvent返回true即可，当然可以通过ViewGroup的requestDisallowInterceptTouchEevent(false)方法来拦截事件。

  
#总结

## 1.   Activity传递到DecorView的过程

当屏幕发生触摸事件时，首先系统会将该触摸事件分发给Activity,然后Activity通过调用dispatchTouchEvent来分发该触摸事件：

```

	/**被调用去处理屏幕触摸事件，可以通过继承该方法拦截所有的触屏事件，使得他们不被发送给Window。
	* @return boolean retrun ture if this event was consumed.
	*/
	public boolean dispatchTouchEvent(MotionEvent ev){
		if(ev.getAction() == MotionEvent.ACTION_DOWN){
			onUserInteraction();
		}
		
		if(getWindow().superDispatchTouchEvent(ev)){
			return true;
		}
		
		return onTouchEvent(ev);
	}
	
	/**
	* 这是一个空方法，当触屏、按键等事件被分发到当前Activity时，会调用该方法。因此如果我们想要监听上述是事件，	* 那么我们就可以集成这个方法。
	*/
	public void onUserIntercation(){}
	
	
```

> 上面getWindow().superDispatchTouchEvent(ev)最终会调用到DecorView.dispatchTouchEvent（）（DecorView是Window的最顶层ViewGroup，它只有一个子元素LinearLayout，包含通知栏、标题栏、内容）。也就是说最终会调用到ViewGroup.dispatchTouchEvent(ev)。现在我们只需要知道DecorView是一个ViewGroup，所以当Activity将事件分发到DecorView之后，DecorView就会按照ViewGroup的事件分发规则将触摸事件分发出去。如果DecorView及它的所有子View和子ViewGroup都不处理该触摸事件，则getWindow().superDispatchTouchEvent(ev)就会返回false，最终就会执行Activity的onTouchEvent(ev).

```
	/**
	* 当且仅当某个触摸事件没有被消费时，该方法被调用。用户按下的是返回键（可能是虚拟返回按键，也可能是实体按键）
	* 时，则关闭当前Activity。
	* @return boolean Return ture if user press on Back.
	*/
	public boolean onTouchEvent(MotionEvent event){
		if(mWindow.showCloseOnTouch(this, event)){
			finish();
			return true;
		}
		
		return false;
	}
	

```

> onTouchEvent(ev)的代码十分简单，仅仅判断用户按下的是否是返回键，如果是则finish当前Activity，否则返回false。

## 2. 经过上述过程分析，我们已经知道触摸事件是如何从Activity传递到DecorView，那么下面我们在详细分析一下ViewGroup是如何将触摸事件分发到View的。

>首先明确一下我们需要解决的问题：①ViewGroup是如何选择出正确的ViewGroup/View的；②如何规避用户的误操作

> 事件分发到ViewGroup之后，首先会调用ViewGroup的dispatchTouchEvent(ev)方法，该方法的目的就是讲触摸事件分发到目标View。由于该方法代码较长，我们只截取其中关键的一部分
> 
>**touchTarget**以单链表的形式用来记录已经遍历过的childView，mFirstTouchTarget是该链表的头。
>viewGroup的子控件可能为view也可能为viewGroup，这个极其重要，谨记！！！！

一定、一定、一定要多读几遍源码
>
>此外先记着以下结论：
>
调用方法dispatchTransformedTouchEvent()将Touch事件传递给特定的子View。该方法十分重要，在该方法中为一个递归调用，会递归调用dispatchTouchEvent()方法。在dispatchTouchEvent()中如果子View为ViewGroup并且Touch没有被拦截那么递归调用dispatchTouchEvent()，如果子View为View那么就会调用其onTouchEvent()。dispatchTransformedTouchEvent方法如果返回true则表示子View消费掉该事件

>：dispatchTransformedTouchEvent 
>
|return        | 描述           | mFirstTouchTarget  |
| ------------- |:-------------|:-----|
| true      | 事件被消费 | null |
| false      | 事件被忽略      |   非null |

```
	public boolean dispatchTouchEvent(MotionEvent ev){
		
		...
		
		int action = ev.getAction();
		int actionMasked = action & MotionEvent.ACTION_MASK;
		
		boolean handled = false;
		
		final boolean intercepted;
		//代码段1：因为ACTION_DOWN是触摸事件的开始事件，那么只要拦截了ACTION_DOWN事件，则后续事件均不会分发。
		//首先判断是否拦截ACTION_DOWN（事件刚开始时，mFirstTouchTarget 为null）
		//mFirstTouchTarget != null是一个很重要的判断条件，如果某个子View消费了ACTION_DOWN事件，那么mFirstTouchTarget 不等于null，但是如果ACTION_DOWN没有被消费，则mFirstTouchTarget等于null，也就处理不了ACTION_MOVE事件了。
		//从这里我们也可以看出，如果ACTION_DWON事件没有被消费，后续事件都会被拦截
		if (actionMasked == MotionEvent.ACTION_DOWN || mFirstTouchTarget != null) {
			final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
			if (!disallowIntercept) {
				intercepted = onInterceptTouchEvent(ev);
				ev.setAction(action); //restore action in case it was changed
			} else {
				intercepted = false;
			}
		} else {
			intercepted = true;
		}
		
		final boolean canceled = resetCancelNextUpFlag(this)
							|| actionMasked == MotionEvent.ACTION_CANCEL;
		
		//代码段2, 如果没有拦截(如果不拦截ACTION_DOWN，那么后续事件也会分发给TargetView)				
		if (!canceled && !intercepted){

			//ACTION_DOWN事件
			if (actionMasked == MotionEvent.ACTION_DOWN
                        || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN)
                        || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {


                    final int childrenCount = mChildrenCount;
                    if (newTouchTarget == null && childrenCount != 0) {

                        final float x = ev.getX(actionIndex);
                        final float y = ev.getY(actionIndex);

                        // Find a child that can receive the event.
                        // Scan children from front to back.
                        final ArrayList<View> preorderedList = buildOrderedChildList();
                        final boolean customOrder = preorderedList == null
                                && isChildrenDrawingOrderEnabled();

                        final View[] children = mChildren;
                        //遍历childView，找到targetView
                        for (int i = childrenCount - 1; i >= 0; i--) {
                            final int childIndex = customOrder
                                    ? getChildDrawingOrder(childrenCount, i) : i;
                            final View child = (preorderedList == null)
                                    ? children[childIndex] : preorderedList.get(childIndex);

                            // If there is a view that has accessibility focus we want it
                            // to get the event first and if not handled we will perform a
                            // normal dispatch. We may do a double iteration but this is
                            // safer given the timeframe.
                            if (childWithAccessibilityFocus != null) {
                                if (childWithAccessibilityFocus != child) {
                                    continue;
                                }
                                childWithAccessibilityFocus = null;
                                i = childrenCount - 1;
                            }
                            //当前View可见且没有动画时
                            if (!canViewReceivePointerEvents(child)
                                    || !isTransformedTouchPointInView(x, y, child, null)) {
                                ev.setTargetAccessibilityFocus(false);
                                continue;
                            }

                            //查找当前view是否在mFirstTouchTarget这个链表上，如果存在返回target，否则返回null
                            newTouchTarget = getTouchTarget(child);

                            //如果不为空，证明已经找到则退出
                            if (newTouchTarget != null) {
                                // Child is already receiving touch within its bounds.
                                // Give it the new pointer in addition to the ones it is handling.
                                newTouchTarget.pointerIdBits |= idBitsToAssign;
                                break;
                            }

                            resetCancelNextUpFlag(child);

                            //如果当前child消费了该事件，则已经找到targetView
                            if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
                                // Child wants to receive touch within its bounds.
                                mLastTouchDownTime = ev.getDownTime();
                                if (preorderedList != null) {
                                    // childIndex points into presorted list, find original index
                                    for (int j = 0; j < childrenCount; j++) {
                                        if (children[childIndex] == mChildren[j]) {
                                            mLastTouchDownIndex = j;
                                            break;
                                        }
                                    }
                                } else {
                                    mLastTouchDownIndex = childIndex;
                                }
                                mLastTouchDownX = ev.getX();
                                mLastTouchDownY = ev.getY();

                                //该child处理了该事件，则将它添加到mFirstTouchTarget链表
                                newTouchTarget = addTouchTarget(child, idBitsToAssign);

                                //标识该触摸事件已经分发到targetView
                                alreadyDispatchedToNewTouchTarget = true;
                                //已经找到targetView，所以跳出，此时newTouchTarget肯定不是null
                                break;
                            }

                            // The accessibility focus didn't handle the event, so clear
                            // the flag and do a normal dispatch to all children.
                            ev.setTargetAccessibilityFocus(false);
                        }
                        if (preorderedList != null) preorderedList.clear();
                    }

                    //newTouchTarget == null 证明没有任何child消费了该事件，如果此时Aciton == ACTION_DOWN，那么mFirstTouchTarget为空，所以此处的ACTION必不是ACTION_DOWN，可能是ACTION_MOWE。
                    if (newTouchTarget == null && mFirstTouchTarget != null) {
                        // Did not find a child to receive the event.
                        // Assign the pointer to the least recently added target.
                        newTouchTarget = mFirstTouchTarget;
                        //将newTouchTarget指向最开始添加到链表中的child。
                        while (newTouchTarget.next != null) {
                            newTouchTarget = newTouchTarget.next;
                        }
                        newTouchTarget.pointerIdBits |= idBitsToAssign;
                    }
                }
		}

		//mFirstTouchTarget == null只有两种情况：①Action == ACTION_DOWN && childCount == 0 ②自己拦截了Action事件
		// Dispatch to touch targets.
        if (mFirstTouchTarget == null) {
                // No touch targets so treat this as an ordinary view.
                // 该触屏事件不在viewGroup的子View上，则将该事件分发给自己
                handled = dispatchTransformedTouchEvent(ev, canceled, null,
                        TouchTarget.ALL_POINTER_IDS);
        } else {
        		// 
                // Dispatch to touch targets, excluding the new touch target if we already
                // dispatched to it.  Cancel touch targets if necessary.
                TouchTarget predecessor = null;
                TouchTarget target = mFirstTouchTarget;
                while (target != null) {
                    final TouchTarget next = target.next;
                    if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {
                        handled = true;
                    } else {
                        final boolean cancelChild = resetCancelNextUpFlag(target.child)
                                || intercepted;
                        if (dispatchTransformedTouchEvent(ev, cancelChild,
                                target.child, target.pointerIdBits)) {
                            handled = true;
                        }
                        if (cancelChild) {
                            if (predecessor == null) {
                                mFirstTouchTarget = next;
                            } else {
                                predecessor.next = next;
                            }
                            target.recycle();
                            target = next;
                            continue;
                        }
                    }
                    predecessor = target;
                    target = next;
                }
        }
	}



	/**
     * Returns true if a child view can receive pointer events.
     */
	 private static boolean canViewReceivePointerEvents(View child) {
        return (child.mViewFlags & VISIBILITY_MASK) == VISIBLE //view可见
                || child.getAnimation() != null;//view没有动画正在运行
    }

    /**
     * Adds a touch target for specified child to the beginning of the list.
     * Assumes the target child is not already present.
     */
    private TouchTarget addTouchTarget(View child, int pointerIdBits) {
        TouchTarget target = TouchTarget.obtain(child, pointerIdBits);
        target.next = mFirstTouchTarget;
        mFirstTouchTarget = target;
        return target;
    }
```

> 根据上面的源码，我们知道ViewGroup的事件分发流程大体如下：

> **ACTION_DOWN**：
1. 判断是否要拦截事件
2. 遍历child，找到要消费事件的child，并将其分发下去。
3. 如果查找到的child是View，则调用view.dispatchTouchEvent并return true,如果是ViewGroup，则回到 1

> **ACTION_MOVE**
1. 之前是否拦截了ACTION_DOWN
2. 如果拦截了ACTION_DOWN，则自己处理该事件
3. 如果没有拦截，则逐个遍历child，找到要消费该事件的child，并将其分发下去
4. 如果查找到的child是view，则分发到view，并return true，如果是ViewGroup则回到 1

> **ACTION_UP**
1. 之前是否拦截ACTION_DOWN
2. 如果拦截了ACTION_DOWN,则自己消费该事件
3. 如果没有拦截，则遍历child，找到要消费该事件的child，并将其分发下去
4. 如果查找到的child是view，则将Action_UP分发下去，并反会true，如果是viewGroup则返回 1

3. 上面介绍了ViewGroup是如何将触摸事件分发到View的，下面我们再来分析一下view是如何处理这些触摸事件的。


> 根据上面介绍，ViewGroup将触摸事件分发到view时，会调用view.dispatchTouchEvent,该函数具体如下：

```
	public boolean dispatchTouchEvent(MotionEvent event){
		...
		
		if (onFilterTouchEventForSecurity(event)) {
            //noinspection SimplifiableIfStatement
            
            ListenerInfo li = mListenerInfo;
            
            if (li != null && li.mOnTouchListener != null //是否设置了mOnTouchListner
                    && (mViewFlags & ENABLED_MASK) == ENABLED //该控件是否enable（可操作）
                    && li.mOnTouchListener.onTouch(this, event)) //onTouch的返回值 {
                result = true;
            }

            if (!result && onTouchEvent(event)) {
                result = true;
            }
        }
        
        ...
	}
```
> 这段代码中的变量mOnTouchListener是通过view.setOnTouchListner(OnTouchListener otl)来设置的，该函数具体如下：

```
	Button button1 = (Button) findViewById(R.id.button1);
	
	/**
     * Called when a touch event is dispatched to a view. This allows listeners to
     * get a chance to respond before the target view.
     *
     * @param v The view the touch event has been dispatched to.
     * @param event The MotionEvent object containing full information about
     *        the event.
     * @return True if the listener has consumed the event, false otherwise.
     */
	button1.setOnTouchListener(new OnTouchListener(){
		public boolean onTouch(View v, MotionEvent event){
			//do something what you want.
			
			return ture; //or false.
		}
	});
```

>我们现在再来看一下view.dispatchTouchEvent代码，view.dispatchTouchEvent的逻辑很简单：如果设置了mOnTouchListener，且该控件是enable的，且onTouch返回ture，那么直接返回ture；在这里需要解释一下：**在onTouch方法中如果返回true，则表明该view消费了该触摸事件，所以就不会再去调用onTouchEvent方法了；如果返回false，则表明没有消费该事件，这时就会去调用onTouchEvent方法。**

```
	public boolean onTouchEvent(MotionEvent event){
		...
		
		if ((viewFlags & ENABLED_MASK) == DISABLED) {
            if (action == MotionEvent.ACTION_UP && (mPrivateFlags & PFLAG_PRESSED) != 0) {
                setPressed(false);
            }
            // A disabled view that is clickable still consumes the touch
            // events, it just doesn't respond to them.
            return (((viewFlags & CLICKABLE) == CLICKABLE
                    || (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE)
                    || (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE);
        }

        if (mTouchDelegate != null) {
            if (mTouchDelegate.onTouchEvent(event)) {
                return true;
            }
        }

        if (((viewFlags & CLICKABLE) == CLICKABLE ||
                (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE) ||
                (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE) {

        	//comsume the touch event
        	
        	return true;
        }
        
        return false;
        
        
	}
```
>我们一段段的来分析这个方法：
>
1. 注意第4-13行，其中有一句注释A disabled view that is clickable still consumes the touch  events, it just doesn't respond to them.意思是说尽管一个view是disabled的，但是如果他是clickable，longClickable，contextClickable的，他仍然会消费该事件，只是不作出响应而已。
2. 第15-19行，检查是否设置了delegate，如果设置了delegate那么该触摸事件由delegate消费。
3. 第21-28行， 就是最主要的逻辑了。如果view是clickable、long_clickable、context_clickable的，那么则自己消费该事件。
4. 第30行，返回false，标识该view不消费该触摸事件。

我们再来看看view.dispatchTouchEvent,第9行的

```
    if (li != null && li.mOnTouchListener != null //是否设置了mOnTouchListner
                    && (mViewFlags & ENABLED_MASK) == ENABLED //该控件是否enable（可操作）
                    && li.mOnTouchListener.onTouch(this, event)) //onTouch的返回值，
```
最后一个条件li.mOnTouchListener.onTouch(this, event),这时我们可知onTouch返回false，则会去调用onTouchEvent，而view的click事件也是在onTouchEvent中调用的，由此可知各个响应事件的响应顺序为：
1. onTouch() return false,则调用onTouchEvent
2. onTouchEvent

到此为止，对于触摸事件分发，我们已经分析完毕了，接下来我们通过一个活动图来做最终的总结。

![](http://i.imgur.com/VIyY6mX.png)


----------