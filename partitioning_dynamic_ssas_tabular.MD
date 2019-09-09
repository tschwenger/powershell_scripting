# script for dynamic partitioning of a fact table

     $DatabaseName= "ASDatabaseName" # enter AS databasename here

     $SourceServerName="ADWDatabaseName" # enter ADW databasename here
     $SourceSchemaName=@("source_schema")  # enter source ADW schema here

     $sourceschemanamefacttable = #variable for source schema and fact table

     $DateColumn = @("[datekeycolumn]") # enter date key column here for fact table
     #Change the below 3 parameters for each environment
     $ServerName = "servername" #AAS Server Name
     $ADWDatabaseName = "adwdatabasename"                                    #Adw database name
     $ADWServerName ="adwservername"          #Adw Server Name

     #Change the credentials for each environment
     #Script to connect to Azure Analysis Service

     $credential = 'credential' #credential to for service account

     $cred = Get-AutomationPSCredential -Name $credential

     # this is a dmv query that returns all the names of the tables for a particular SSAS Model 
     $getASmodeltablenames="SELECT Table_Name FROM `$System`.DBSCHEMA_TABLES  WHERE TABLE_TYPE = 'SYSTEM TABLE'"

     # this is a powershell command that runs the above query against the specified variables for the server, database and query
     # the results of this command return the table names in an unuseful xmls format that needs further manipulation
     $queryresults = Invoke-ASCmd -Server $ServerName -Database $DatabaseName  -Query $getASmodeltablenames -Credential $cred


     # this get us the starting point of the array we want to pass through a powershell command in later steps
     # the start of the array is basically a table name in the model
     # this is the beginning of the substring we want to manipulate
     $startofindex = $queryresults.indexof("<Table_Name>")+12

     # this finds the end of the substring we want to get
     $endofindex = $queryresults.indexof("</Table_Name></row></root></return>")


     # the length of this substring is what contains the model's table names in xmla format
     $substringlength = $endofindex - $startofindex

     # this variable yields the substring we are going to clean
     $tablenamesxmlaresult = $queryresults.substring($startofindex,$substringlength)

     # this variable replaces the xmla junk and replaces it with a comma for the array we want to create

     $tablenames =$tablenamesxmlaresult -replace("</Table_Name></row><row xmlns=`"urn:schemas-microsoft-com:xml-analysis:rowset`"><Table_Name>","," )

     # this converts the substring into and array by the comma delimiter
     $arraytablenames = $tablenames.split(",")

     # this creates a dimension array by giving us all the dimension tables
     $Dimensiontablearray = $arraytablenames | WHERE-object {($_ -notlike 'Fact*')}

     # this creates a fact table array by giving us all the fact tables in one array
     $Facttablearray = $arraytablenames | WHERE-object {($_ -like 'Fact*')}


     # this variable takes the dimension array and passes it through the invoke process table command that will process all off the dimensions in our model

     foreach ($Dimensiontablearray in $Dimensiontablearray)

     {Invoke-ProcessTable -Credential $cred -Server $ServerName -DatabaseName $DatabaseName -RefreshType "Full" -TableName $Dimensiontablearray  }


     #process fact tables
     # the split function turns a single string into an array
     $TableName= -split $Facttablearray
     $SourceTableName=-split $Facttablearray

     $j=-1

     #Script to create partitions for the model

     While ($j -le 0)
     {


     for ($i = 0; $i -le ($TableName.length - 1); $i += 1)
      {

     $CurrentTableName = $TableName[$i]
     $CurrentSourceSchemaName =$SourceSchemaName[$i]
     $CurrentSourceTableName = $SourceTableName[$i]
     $CurrentDateColumn = $DateColumn[$i]

     $x =(get-date).AddMonths($j)

     $MonthStart =$x.ToString("yyyy, MM, ")+ "01, 0, 0, 0"

     $MonthEnd=$x.ToString("yyyy, MM, ")+ [DateTime]::DaysInMonth((get-date).Year, (get-date).Month+$j) +", 0, 0, 0"

     $PartitionName = $CurrentTableName+$x.ToString("yyyyMM")

     $TMSLScript = " 
     {
       `"createOrReplace`": {
         `"object`": {
           `"database`": `"$DatabaseName`",
           `"table`": `"$CurrentTableName`",
           `"partition`": `"$PartitionName`"
         },
         `"partition`": {
           `"name`": `"$PartitionName`",
           `"source`": {
             `"type`": `"m`",
             `"expression`": [
               `"let`",
               `"    Source = #\`"$SourceServerName\`",`",
               `"    $CurrentSourceTableName = Source{[Schema=\`"$CurrentSourceSchemaName\`",Item=\`"$CurrentSourceTableName\`"]}[Data],`",
               `"    #\`"Filtered Rows\`" = Table.SelectRows($CurrentSourceTableName, each $CurrentDateColumn >= #datetime($MonthStart) and $CurrentDateColumn  <= #datetime($MonthEnd))`",
               `"in`",
               `"    #\`"Filtered Rows\`"`"
             ]
           },      
         }
       }
     }

     "
     #command to create partitions
     Invoke-ASCmd -Server $ServerName  -Query $TMSLScript -Credential $cred

     #Command to Process Add for above partitions

     $FactTableProcessData = "

     {
       `"refresh`": {
         `"type`": `"full`",
         `"objects`": [
           {
             `"database`": `"$DatabaseName`",
             `"table`": `"$CurrentTableName`",
             `"partition`": `"$PartitionName`"
           }
         ]
       }
     }

     "
     Invoke-ASCmd -Server $ServerName  -Query $FactTableProcessData -Credential $cred

     }

     $j=$j+1
     }

     #ProcessAdd for FactInventory

     $DaxQuery = "Evaluate ROW(`"Date`",Max($DateColumn))"
     $XMLResultSet = Invoke-ASCmd -Server $ServerName -Database $DatabaseName  -Query $DaxQuery -Credential $cred 
     $StartPosition=$XMLResultSet.INDEXOF("<_x005B_Date_x005D_>")+20
     $EndPosition=$XMLResultSet.INDEXOF("</_x005B_Date_x005D_>")
     $Length=$EndPosition-$StartPosition
     $MaxSnapshotDate = $XMLResultSet.Substring($StartPosition,$Length)
     $MaxSnapshotDate = "'$MaxSnapshotDate'"

     <#
     $TempCurrentDate = Get-Date
     $PartitionYear   = $TempCurrentDate.Year
     $ProcessAddPartitionName = $TableName +$PartitionYear 
     #>



     $ProcessAdd = 
     "
     {
     `"refresh`":{
     `"type`":`"add`",
     `"objects`":[
     {
     `"database`":`"$DatabaseName`",
     `"table`":`"$CurrentTableName`",
     `"partition`":`"$ProcessAddPartitionName`"
     }
     ],
     `"overrides`":[
     {
     `"partitions`":[
     {
     `"originalObject`":{
     `"database`":`"$DatabaseName`",
     `"table`":`"$CurrentTableName`",
     `"partition`":`"$ProcessAddPartitionName`"
     },
     `"source`":{
     `"type`": `"m`",
     `"expression`": [

     `"let x=Sql.Database(\`"$ADWServerName\`",\`"$ADWDatabaseName\`",[Query=\`"select * from (sourceschemafacttable) where DateKeyColumn>$MaxSnapshotDate \`"]) in x`",
     ],
     }
     }
     ]
     }
     ]
     }
     }

     "
     Invoke-ASCmd -Server $ServerName  -Query $ProcessAdd  -Credential $cred

     # Script to Recalc the ProductPerformance Model
     # need the recalc statement because of calculated columns and the last refresh date measure

     $ProcessRecalc = "
     {
       `"refresh`": {
         `"type`": `"calculate`",
         `"objects`": [
           {
             `"database`": `"$DatabaseName`"
           }
         ]
       }
     }

     "

     Invoke-ASCmd -Server $ServerName  -Query $ProcessRecalc  -Credential $cred
