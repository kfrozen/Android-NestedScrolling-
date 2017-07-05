## Android中关于NestedScrolling的那些事 ##

这里分为两部分，Parent和Child，其实是两组接口，可以根据具体需求去实现。例如RecyclerView只是实现了NestedScrollingChild，也即RecyclerView不支持再嵌套NestedScrolling的子View，而NestedScrollView则同时实现了两组接口，表明其既可以作为Child被嵌套也可以作为Parent嵌套别的NestScrolling的View。

如果存在嵌套，事件是由最里层的ChildView向外层的ParentView逐层分发，作为中间层的View，既要接受子View传来的事件进行选择性处理，还要负责继续进行分发。这里还是先列出一个事件周期内各主要方法被调用的顺序(P代表NestedScrollingParent的方法，C代表NestedScrollingChild的方法)：

onStartNestedScroll(C) --> onStartNestedScroll(P) --> dispatchNestedPreScroll(C) --> onNestedPreScroll(P) --> dispatchNestedScroll(C) --> onNestedScroll(P) --> [(if has Fling action) dispatchNestedPreFling(C) --> onNestedPreFling(P) --> dispatchNestedFling(C) --> onNestedFling(P)] --> onStopNestedScroll(C) --> onStopNestedScroll(P)

PS: 上述顺序仅基于一层嵌套，即一个ScrollingParent嵌套一个ScrollingChild，若牵扯多层嵌套，则中间层的View在处理onNestedXXX的同时需要完成dispatchNestedXXX。


----------


- **NestedScrollingParent**

		
		public boolean onStartNestedScroll(View child, View target, int nestedScrollAxes);
		--这里确定该Parent是否要响应子View的Nested Scrolling事件
		
		public void onNestedPreScroll(View target, int dx, int dy, int[] consumed);
		--在滚动开始前会被调用，在此Parent有机会选择消费一部分事件，最后一个参数consumed是输出，参考下面的代码：
		public boolean dispatchNestedPreScroll(int dx, int dy, int[] consumed, int[] offsetInWindow) {
				
			......		
	
				......
				
                if (consumed == null) {
                    if (mTempNestedScrollConsumed == null) {
                        mTempNestedScrollConsumed = new int[2];
                    }
                    consumed = mTempNestedScrollConsumed;
                }
                consumed[0] = 0;
                consumed[1] = 0;
                ViewParentCompat.onNestedPreScroll(mNestedScrollingParent, mView, dx, dy, consumed);

				.......

            	return consumed[0] != 0 || consumed[1] != 0;

			......
    	}
		可以看出Parent的onNestedPreScroll方法是在Child的dispatchNestedPreScroll中被调用的，且Child的dispatchNestedPreScroll方法的返回值决定于consumed数组的值，如果Parent进行了消费，则该方法会返回true。

		public void onNestedScroll(View target, int dxConsumed, int dyConsumed, int dxUnconsumed, int dyUnconsumed);
		--在滚动结束之后该方法会被调用，子View消费的和未被消费的事件都会被传递至此。

----------

- **NestedScrollingChild**

		public boolean startNestedScroll(int axes);
		--这里主要是在该View的ParentViews中寻找实现了NestedScrollingParent接口的Parent，并且确定nested scrolling事件是否已经开始，若存在这样的Parent并nested scrolling事件已成功开始，则返回true。

		public boolean dispatchNestedPreScroll(int dx, int dy, int[] consumed, int[] offsetInWindow)
		--在滚动开始前，ChildView在这里将事件分发给可以接收该事件的ParentView，返回true表示Parent消费了该事件，false则反之，该返回值取决于ViewParentCompat.onNestedPreScroll方法中对于consumed数组的改动。

		public boolean dispatchNestedScroll(int dxConsumed, int dyConsumed, int dxUnconsumed, int dyUnconsumed, int[] offsetInWindow) 
		--同样的，滚动完成后ChildView在这里将事件分发，该方法中会调用 ViewParentCompat.onNestedScroll(...)，如果Parent有消费，则返回true，反之返回false。

需要注意的是，滑动(Fling)手势依旧会先触发上述提到的Scroll相关事件，之后才会进入到Fling相关的方法中。

-----------

- **CoordinateLayout下双向滚动嵌套情形的注意事项**

这两天碰到一个普遍的issue，当一个以CoordinateLayout为root view的页面中含有双向滚动嵌套（纵向的RecyclerView或者NestedScrollView中嵌套横向RecyclerView）时，作用在横向RecyclerView范围内的纵向滚动事件不会被CoordinateLayout响应，也即很难将AppBarLayout推上去或拉下来。下面就先来分析一下出现这种情况的原因：

先看一下RecyclerView的onInterceptTouchEvent方法的部分源码（NestedScrollView类似）：

	@Override
    public boolean onInterceptTouchEvent(MotionEvent e) {
    	
    	......
      
      //从LayoutManager中拿到滚动方向	
      final boolean canScrollHorizontally = mLayout.canScrollHorizontally();
      final boolean canScrollVertically = mLayout.canScrollVertically();
      
      ......
      
      switch (action) {
            case MotionEvent.ACTION_DOWN:
                if (mIgnoreMotionEventTillDown) {
                    mIgnoreMotionEventTillDown = false;
                }
                mScrollPointerId = e.getPointerId(0);
                mInitialTouchX = mLastTouchX = (int) (e.getX() + 0.5f);
                mInitialTouchY = mLastTouchY = (int) (e.getY() + 0.5f);

                if (mScrollState == SCROLL_STATE_SETTLING) {
                    getParent().requestDisallowInterceptTouchEvent(true);
                    setScrollState(SCROLL_STATE_DRAGGING);
                }

                // Clear the nested offsets
                mNestedOffsets[0] = mNestedOffsets[1] = 0;

                int nestedScrollAxis = ViewCompat.SCROLL_AXIS_NONE;
                if (canScrollHorizontally) {
                    nestedScrollAxis |= ViewCompat.SCROLL_AXIS_HORIZONTAL;
                }
                if (canScrollVertically) {
                    nestedScrollAxis |= ViewCompat.SCROLL_AXIS_VERTICAL;
                }
                
                //根据滚动方向发起一个nestedScroll事件，这里是关键，后面会继续说
                startNestedScroll(nestedScrollAxis);
                break;
                
            case MotionEvent.ACTION_MOVE: {
                final int index = e.findPointerIndex(mScrollPointerId);
                
                /*
                * 一个题外话，这里是拿到触发此次event的pointer，这个pointer是在ACTION_DOWN被触发的时候记录的，
                * 因此如果一个事件的ACTION_DOWN没有被处理而后续MOVE事件被要求处理时，就会被拒绝，直接返回false
                */
                if (index < 0) {
                    Log.e(TAG, "Error processing scroll; pointer index for id " +
                            mScrollPointerId + " not found. Did any MotionEvents get skipped?");
                    return false;
                }

                final int x = (int) (e.getX(index) + 0.5f);
                final int y = (int) (e.getY(index) + 0.5f);
                if (mScrollState != SCROLL_STATE_DRAGGING) {
                    final int dx = x - mInitialTouchX;
                    final int dy = y - mInitialTouchY;
                    boolean startScroll = false;
                    
                    //根据LayoutManager中设定的滚动方向以及事件手势的方向决定是不是进入滚动状态
                    if (canScrollHorizontally && Math.abs(dx) > mTouchSlop) {
                        mLastTouchX = mInitialTouchX + mTouchSlop * (dx < 0 ? -1 : 1);
                        startScroll = true;
                    }
                    if (canScrollVertically && Math.abs(dy) > mTouchSlop) {
                        mLastTouchY = mInitialTouchY + mTouchSlop * (dy < 0 ? -1 : 1);
                        startScroll = true;
                    }
                    if (startScroll) {
                    	 //进入滚动状态
                        setScrollState(SCROLL_STATE_DRAGGING);
                    }
                }
            } break;
            
         }
         
        //实际效果就是当事件处于滚动状态时，拦截掉事件 
        return mScrollState == SCROLL_STATE_DRAGGING;
    }
    
从上面的代码可以看出，RecyclerView对于DOWN事件会进行处理，但永远不会拦截，因此DOWN事件总会向下继续传递，而对于MOVE事件，如果事件的方向与RecyclerView的方向一致，则会被拦截，不再向下传递。因此如果是一个垂直的RecyclerView嵌套横向的RecyclerView，垂直方向的滚动事件会被充当parent的垂直RecyclerView截获，对于充当child的横向RecyclerView只会收到一个DOWN事件和一个CANCEL事件。可以看到RecyclerView或NestedScrollView将对event的处理前移到了onInterceptTouchEvent中。

接着往下看，在对DOWN处理的过程中，最后一步会调用startNestedScroll(nestedScrollAxis)， 来看源码：

	public boolean startNestedScroll(int axes) {
		  //如果之前已经有过判断并且nested scroll事件已经在进行，则直接返回true
        if (hasNestedScrollingParent()) {
            // Already in progress
            return true;
        }
        
        //判断该View是否enable了NestedScrolling功能，方法默认返回true，这个方法是解决上面提到的issue的关键
        if (isNestedScrollingEnabled()) {
            ViewParent p = mView.getParent();
            View child = mView;
            while (p != null) {
            	//遍历所有的parent view，并通过onStartNestedScroll询问其是否愿意处理此View发出的NestedScrolling事件
                if (ViewParentCompat.onStartNestedScroll(p, child, mView, axes)) {
                    mNestedScrollingParent = p;
                    ViewParentCompat.onNestedScrollAccepted(p, child, mView, axes);
                    
                    //如果某个parent view接受了，则返回true
                    return true;
                }
                if (p instanceof View) {
                    child = (View) p;
                }
                p = p.getParent();
            }
        }
        
        //不支持NestedScrolling或没有parent愿意处理自己发出的NestedScrolling事件，返回false
        return false;
    }
    
可以看到如果支持NestedScrolling，则会去遍历其父view并调用onStartNestedScroll，这里先列出NestedScrollView和CoordinateLayout这两个最常用的NestedScrollingParent的onStartNestedScroll方法源码：

1. NestedScrollView：

	    @Override
	    public boolean onStartNestedScroll(View child, View target, int nestedScrollAxes) {
	        return (nestedScrollAxes & ViewCompat.SCROLL_AXIS_VERTICAL) != 0;
	    }
	    
2. CoordinateLayout（CoordinateLayout最终会调用其viewBehavior的同名方法，因此直接列出behavior中的）：

		**AppBarLayout.Behavior.onStartNestedScroll：
		
		@Override
        public boolean onStartNestedScroll(CoordinatorLayout parent, AppBarLayout child,
                View directTargetChild, View target, int nestedScrollAxes) {
            // Return true if we're nested scrolling vertically, and we have scrollable children
            // and the scrolling view is big enough to scroll
            final boolean started = (nestedScrollAxes & ViewCompat.SCROLL_AXIS_VERTICAL) != 0
                    && child.hasScrollableChildren()
                    && parent.getHeight() - directTargetChild.getHeight() <= child.getHeight();

            if (started && mOffsetAnimator != null) {
                // Cancel any offset animation
                mOffsetAnimator.cancel();
            }

            // A new nested scroll has started so clear out the previous ref
            mLastNestedScrollingChildRef = null;

            return started;
        }
        
看这一句：(nestedScrollAxes & ViewCompat.SCROLL_AXIS_VERTICAL) != 0，说明这两个NestedScrollingParent都是只接收垂直方向的事件，**因此横向的RecyclerView发出的startNestedScroll请求一定会返回false，而被调用到onStartNestedScroll的parent views也都会停止接收NestedScrolling事件，会直接调用onStopNestedScroll方法。**

这就是导致上述的issue的原因，一个垂直的滚动事件会先触发parent层的垂直RecyclerView的onInterceptTouchEvent，在DOWN的处理中会**第一次**调用startNestedScroll，这时上层的NestedScrollingParent会正常响应，但由于DOWN事件不被拦截，child层的横向RecyclerView也会接收到这个DOWN，并且**第二次**调用startNestedScroll，这一次NestedScrollingParent由于方向不对都不会接收并调用onStopNestedScroll，这样一来，第一次成功的dispatch被覆盖，整个NestedScrolling事件就不会被响应。

**解决办法：**

对横向的RecyclerView设置setNestedScrollingEnabled(false)。来看为什么，回到RecyclerView的startNestedScroll方法中，看到有这么一个判断条件：
	
	if (isNestedScrollingEnabled())
	
如果我们将其设为false，则方法会直接返回false，而不会调用其parent的onStartNestedScroll方法，从而不会覆盖第一次成功的dispatch。相当于横向RecyclerView发出的所有NestedScrollling事件会在它自己这一层被拦下，不会被dispatch出去。

		
