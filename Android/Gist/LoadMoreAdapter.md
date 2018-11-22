```kotlin
import android.content.Context
import android.support.v7.widget.LinearLayoutManager
import android.support.v7.widget.RecyclerView
import android.view.View
import android.view.ViewGroup
import java.util.ArrayList

abstract class RecyclerViewHolder<T>(itemView: View) : RecyclerView.ViewHolder(itemView) {

    protected val context: Context
        get() = itemView.context

    abstract fun bindView(item: T)
}

abstract class FooterLoadMore<T>(itemView: View): RecyclerViewHolder<T>(itemView) {

    override fun bindView(item: T) {}
    abstract fun showState(state: Int, message: String?)
}

abstract class LoadMoreRecyclerAdapter<T>(val context: Context)
    : RecyclerView.Adapter<RecyclerViewHolder<T>>() {

    companion object {
        const val TYPE_DATA_ITEM = 0
        const val TYPE_FOOTER_MORE = 1

        const val FOOTER_COUNT = 1

        const val FOOTER_IDLE = 0
        const val FOOTER_LOADING = 1
        const val FOOTER_COMPLETE = 2
        const val FOOTER_ERROR = 3
        const val FOOTER_NO_MORE = 4
    }

    private var loadState = FOOTER_IDLE
    private var loadMessage: String? = null
    private val items = ArrayList<T>()

    var itemClicker: ((item: T) -> Unit)? = null
    var loadMoreListener: (() -> Unit)? = null

    val originItems: MutableList<T>
        get() = items

    abstract fun createDataViewHolder(parent: ViewGroup): RecyclerViewHolder<T>
    abstract fun createFooterViewHolder(parent: ViewGroup): FooterLoadMore<T>
    open fun showFooterForState(state: Int): Boolean = false
    open fun disallowLoadMore(state: Int): Boolean = false

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): RecyclerViewHolder<T> {
        if (viewType == TYPE_FOOTER_MORE) {
            return createFooterViewHolder(parent)
        }
        return createDataViewHolder(parent)
    }

    override fun onBindViewHolder(holder: RecyclerViewHolder<T>, position: Int) {
        val item = items[position]
        holder.bindView(item)
        if (itemClicker != null) {
            holder.itemView.setOnClickListener { itemClicker?.invoke(item) }
        }
    }

    override fun getItemViewType(position: Int): Int {
        if (position == originItems.size && hasFooter()) {
            return TYPE_FOOTER_MORE
        }
        return TYPE_DATA_ITEM
    }

    private fun hasFooter(): Boolean {
        return originItems.size > 0 && showFooterForState(loadState)
    }

    override fun getItemCount(): Int {
        return originItems.size + ( if (hasFooter()) FOOTER_COUNT else 0 )
    }

    fun getItems(): List<T> = ArrayList(items)

    fun getItem(index: Int) = items[index]

    fun setItems(items: Collection<T>?) {
        if (items == null) {
            return
        }
        this.items.clear()
        this.items.addAll(items)
        notifyDataSetChanged()
    }

    open fun appendItems(items: Collection<T>?) {
        if (items == null) {
            return
        }
        this.items.addAll(items)
        notifyItemRangeChanged(this.items.size, items.size)
    }

    override fun onAttachedToRecyclerView(recyclerView: RecyclerView) {
        super.onAttachedToRecyclerView(recyclerView)
        recyclerView.addOnScrollListener(scrollListener)
    }

    override fun onDetachedFromRecyclerView(recyclerView: RecyclerView) {
        recyclerView.removeOnScrollListener(scrollListener)
        super.onDetachedFromRecyclerView(recyclerView)
    }

    private val scrollListener = object: RecyclerView.OnScrollListener() {

        private var lastVisibleItem = 0

        override fun onScrollStateChanged(recyclerView: RecyclerView, newState: Int) {
            super.onScrollStateChanged(recyclerView, newState)
            val itemCount = recyclerView.adapter?.itemCount ?: 0
            if (newState == RecyclerView.SCROLL_STATE_IDLE
                    && lastVisibleItem + 1 >= itemCount) {
                if (loadState != FOOTER_LOADING && disallowLoadMore(loadState) && itemCount > 0) {
                    loadMoreListener?.invoke()
                }
            }
        }

        override fun onScrolled(recyclerView: RecyclerView, dx: Int, dy: Int) {
            super.onScrolled(recyclerView, dx, dy)
            if (recyclerView.layoutManager is LinearLayoutManager) {
                val layoutManager = recyclerView.layoutManager as LinearLayoutManager
                lastVisibleItem = layoutManager.findLastCompletelyVisibleItemPosition()
            }
        }
    }

    fun updateLoadState(state: Int, message: String? = null) {
        val wasShown = showFooterForState(loadState)
        val willShow = showFooterForState(state)
        loadState = state
        loadMessage = message
        if (wasShown && !willShow) {
            notifyItemRemoved(originItems.size)
        } else if (!wasShown && willShow) {
            notifyItemInserted(originItems.size)
        } else if (willShow) {
            notifyItemChanged(originItems.size)
        }
    }
}
```
