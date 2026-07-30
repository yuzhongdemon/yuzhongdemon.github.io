 <script lang="ts">
   import Icon from "@iconify/svelte";
 
   let currentDate = $state(new Date());
 
   const today = new Date();
   const weekDays = ['日', '一', '二', '三', '四', '五', '六'];
 
   let year = $derived(currentDate.getFullYear());
   let month = $derived(currentDate.getMonth());
 
   const monthNames = ['一月', '二月', '三月', '四月', '五月', '六月',
     '七月', '八月', '九月', '十月', '十一月', '十二月'];
 
   let daysInMonth = $derived(new Date(year, month + 1, 0).getDate());
   let firstDay = $derived(new Date(year, month, 1).getDay());
 
   let calendarDays = $derived.by(() => {
     const days: (number | null)[] = [];
     for (let i = 0; i < firstDay; i++) days.push(null);
     for (let d = 1; d <= daysInMonth; d++) days.push(d);
     return days;
   });
 
   function isToday(day: number): boolean {
     return day === today.getDate() && month === today.getMonth() && year === today.getFullYear();
   }
 
   function prevMonth() {
     currentDate = new Date(year, month - 1, 1);
   }
 
   function nextMonth() {
     currentDate = new Date(year, month + 1, 1);
   }
 </script>
 
 <div class="card-base p-3 pb-2 onload-animation">
   <!-- Header: month/year + nav arrows -->
   <div class="flex items-center justify-between px-1 mb-2">
     <span class="font-bold transition text-base text-neutral-900 dark:text-neutral-100 relative ml-1
       before:w-1 before:h-4 before:rounded-md before:bg-[var(--primary)]
       before:absolute before:-left-3 before:top-[2px]">
       {monthNames[month]} {year}
     </span>
     <div class="flex gap-1">
       <button onclick={prevMonth}
         class="btn-plain rounded-lg h-7 w-7 active:scale-90 flex items-center justify-center"
         aria-label="上个月">
         <Icon icon="material-symbols:chevron-left-rounded" class="text-[1.1rem]"></Icon>
       </button>
       <button onclick={nextMonth}
         class="btn-plain rounded-lg h-7 w-7 active:scale-90 flex items-center justify-center"
         aria-label="下个月">
         <Icon icon="material-symbols:chevron-right-rounded" class="text-[1.1rem]"></Icon>
       </button>
     </div>
   </div>
 
   <!-- Week day headers -->
   <div class="grid grid-cols-7 mb-1">
     {#each weekDays as day}
       <div class="text-center text-xs font-medium transition text-black/40 dark:text-white/40 py-1">
         {day}
       </div>
     {/each}
   </div>
 
   <!-- Calendar grid -->
   <div class="grid grid-cols-7">
     {#each calendarDays as day}
       {#if day !== null}
         <button
           class="text-center text-sm py-1 rounded-lg transition
             {isToday(day)
               ? 'bg-[var(--primary)] text-white font-bold shadow-sm'
               : 'text-neutral-800 dark:text-neutral-200 hover:bg-[var(--btn-regular-bg)] active:bg-[var(--btn-regular-bg-active)]'
             }"
           aria-label="{year}年{month + 1}月{day}日"
         >
           {day}
         </button>
       {:else}
         <div class="py-1"></div>
       {/if}
     {/each}
   </div>
 </div>
