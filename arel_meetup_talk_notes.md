I. What is Arel  
  * A RElational aLgebra (best guess on acronym)  
  * Abstract Syntax Tree (AST) that AR relies on to translate to SQL

II. What can Arel do that AR can’t  

Main reason I like it: make join types, selects, aliases completely explicit and predictable  


  * Provide SQL joins not supported by AR (used to be left_outer_joins weren’t supported)  
  * convenient built-ins: gt, lt, lteq, between
  * Define operations that must be done in raw sql at present  
    Examples:  
    - polymorphic outer join  
    - define subquery  
    - cast return values  
    - perform calculations (COALESCE)  

  * make queries composable, parameterizable, easier to test
  * avoid raw SQL & string manipulation  
  * syntax validation/linting
  * MAIN APPEAL: expose these operations as ruby method calls  
    * more explicit and readable 
    * reveals intentions   
  * CLOSE SECOND: give precise control over how queries are assembled rather than relying on AR to guess right  
    * Example: `includes` (column name collision), `preload` (AR chooses the join)  
    * issues with type of join, losing alias: https://github.com/rails/rails/issues/32701
    * (from TB post): 

```
class ProfileGroupMemberships < Struct.new(:user, :visitor)  
  
  def groups  
    @groups ||= user.groups.where("private = false OR id IN ?", visitor.group_ids)  
  end 
end  
```  
  "If we join to another table, our query will immediately break due to the ambiguity of the id column. Even if we qualify the columns with the table name, this will break as well if Rails decides to alias the table name."  
  * bonus: must specify columns or use `Arel.star`: prevents automatically grabbing all columns, as `includes` does.
  * push even more db-specific logic down to db, which has been optimized for it
  * (related) reduce n+1s, especially when it comes to formatting selected values (especially especially when using PG jsonb data type)  

CLOSE SECOND: expose these operations as ruby method calls  

  * more explicit and readable  
  * reveals intentions  

Provide SQL joins not supported by AR (used to be left_outer_joins weren’t supported)  

  * convenient built-ins: gt, lt, lteq, between
  * Define operations that must be done in raw sql at present  
    Examples:  
    - polymorphic outer join  
    - define subquery  
    - cast return values  
    - perform calculations (COALESCE)  

  * make composable queries easier to write, test, debug  
  * avoid raw SQL & string manipulation  

III. Cons  

  * std Rails core team answer: it's a private API, not really meant to be exposed
    *  "However, as it is a private API, you should be cautioned that using it has a cost of potential breaking changes should you upgrade Rails; Arel seems to change its major version each Rails release."
  * Still an issue: https://github.com/rails/rails/issues/36761#issuecomment-515286636
  * (related): not great documentation
  * Lerning curve, whether devs are used to raw SQL or not accustomed to writing anything very complex in SQL 
  * Preference for ruby methods vs. raw SQL, even at expense of setup, is pretty subjective
  * Rails crew has a history of adding the really useful things ... *eventually*  
    *   `.not` (as in `where.not( ... )`
    *  `.or`
    *  `.left_outer_joins`

IV. Good Use Cases  

  *  modules that define frequently-used but complex joins  
  *  with form objects: controllers that receive requests with variable selectors, ordering clauses, where filters over substantially similar joins:  
    *  implement query builders to compose queries dynamically
  *  defining frequently-reference values (booleans, sums, max, etc) that require calculation/comparison across tables or cols
    *  Example: (use COALESCE)
    *  Example: (use 2 subqueries with EXISTS)
    *  Example: (from `iRonin.it`): fname, lname, users with nil in one or the other
    *  time-range scopes: https://github.com/rails/rails/issues/36761#issuecomment-520876203

V. Detail  
  I.  `Model.arel_table`: gives you all the arel methods & `SelectManager` stuff  
  II. Using AR model means it has db connection info already. Arel doesn't know about db config.  
  III. outer joins: pass `Arel::Nodes::OuterJoin`  
    -- where composability becomes useful: a common outer join can be stored in a method:  


    def Artist < ActiveModel::Base  
    
     has_many :artist_roles  
     has_many :roles, through: :artist_roles  
    
    class << self  
    
      def artist_t  
      @_artist_t ||= Artist.arel_table
      end

      def artist_featured  
        Artist.join(:roles, Arel::Nodes::OuterJoin).
          on(artist_t[:id].eq(role_t[:artist_id]).
          where(role_t[:role_name].eq('featured'))
      end
        
    end  

  -- Of course, Rails 5 finally gave us `left_outer_join`  
  
  VI. Rails 6: Arel is no longer a separate gem; gonna get more public support
    *  https://github.com/rails/rails/issues/36761#issuecomment-520876203
 
### Other resources
official docs: https://www.rubydoc.info/github/rails/arel/Arel/Predications

Great write-up (even after 6 yrs): https://jpospisil.com/2014/06/16/the-definitive-guide-to-arel-the-sql-manager-for-ruby.html

Classic: http://radar.oreilly.com/2014/05/more-than-enough-arel.html

Useful (but beware of deprecated `Arel::Table.new`, but still useful: https://devhints.io/arel

Wikipedia for AST: https://en.wikipedia.org/wiki/Abstract_syntax_tree

Wikipedia Visitor Pattern: https://en.wikipedia.org/wiki/Visitor_pattern

https://blog.codeship.com/creating-advanced-active-record-db-queries-arel/  

https://thoughtbot.com/blog/using-arel-to-compose-sql-queries

https://www.ironin.it/blog/intro-to-arel-the-database-agnostic-sql.html

Former coworker/tech-lead with an excellent walk-through: https://dev.to/sophiedebenedetto/composable-query-builders-in-rails-with-arel-31aj

Specific refactoring example: https://medium.com/@zedalaye/build-crazy-queries-with-activerecord-and-arel-e7bd3b606324

For refactoring practice: https://blog.bigbinary.com/2020/01/07/rails-multiple-polymorphic-joins.html

https://www.rubydoc.info/github/rails/arel/Arel/

