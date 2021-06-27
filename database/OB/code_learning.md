# optimizer

key functions:

- `optimize(stmt, logical_plan)`
    - `ObLogPlanFactory::create(ctx, stmt)`
    - `ObSelectLogPlan::generate_plan()`
        - `ObSelectLogPlan::generate_raw_plan()`
            - `ObSelectLogPlan::generate_plan_for_plain_select()`
                - `ObLogPlan::generate_plan_tree()`
                    - `right_join_to_left(stmt)`
                    - `ObLogPlan::generate_join_orders()`
                        - `ObLogPlan::generate_base_level_join_order`
                        - `ObJoinOrder::generate_base_paths()`
                            - `ObJoinOrder::generate_normal_access_path()`
                                - `ObJoinOrder::generate_access_paths(PathHelper& helper)`
                                    - `ObJoinOrder::add_table(tbl_id, ..., access_paths)`: find all access paths of this table
                                        - `get_valid_index_idx(tbl_id, ..., valid_index_ids)`: get all indexes of this table
                                        - `fill_index_info_cache(...)`: get all index information by index IDs
                                        - `add_table_by_heuristics(...)`: use some heuristics rule to choose an access path
                                            - `user_table_heristics`
                                        - `pruning_index(...)`: use skyline pruning to prune some unoptimizal indexes
                                            - `cal_dimension_info`: calculate skyline pruning dimension
                                        - `compute_pruned_index()`: 
                                        - `create_access_path`: create corresponding access path for each available index
                                    - `compute_table_location_for_paths`
                                    - `estimate_size_and_width_for_access`
                                    - `compute_cost_and_prune_access_path`
                    - `ObLogPlan::candi_init()`
                - `ObLogPlan::get_current_best_plan(ObLogicalOperator*& best_plan)`
                - `ObLogPlan::set_final_plan_root(ObLogicalOperator* best_plan)`
        - `plan_traverse_loop`

```cpp
/*
 * Different with cost-based approach, this function tries to use heuristics to generate access path
 * The heuristics used to generate access path is as follows:
 * 1 if search condition cover an unique index key (primary key is treated as unique key), and no need index_back, use
 * that key, if there is multiple such key, choose the one with the minimum index key count 2 if search condition cover
 * an unique index key, need index_back, use that key and follows a refine process to find a better index 3 otherwise
 * generate all the access path and choose access path using cost model
 */
int ObJoinOrder::user_table_heuristics
```