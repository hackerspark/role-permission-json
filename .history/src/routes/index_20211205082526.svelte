<script>
    import XLSX from "xlsx";
  
    const SHEET_NAME = "Roles-Permission-Access";
  
    const FEATURE_NAMES_ROW_INDEX = 0;
    const ACTION_NAMES_ROW_INDEX = 1;
    const DATA_ROW_INDEX = 2;
  
    const ROLE_NAMES_COLUMN_START_INDEX = 0;
    const ROLE_NAMES_COLUMN_END_INDEX = 2;
    const FEATURE_NAMES_COLUMN_START_INDEX = 3;
  
    let rolePermissionMapping = "";
  
    async function transformToMappingJSON({
      target: {
        files: [file]
      }
    }) {
      const sheetJSON = await getSheetJSON(file, SHEET_NAME);
  
      let finalMapping = {};
  
      for (let index = DATA_ROW_INDEX; index < sheetJSON.length; index++) {
        if (!sheetJSON[index].length) {
          continue;
        }
  
        generateRoleFeatureMapping(
          sheetJSON,
          index,
          ROLE_NAMES_COLUMN_START_INDEX,
          ROLE_NAMES_COLUMN_END_INDEX,
          finalMapping
        );
      }
  
      rolePermissionMapping = JSON.stringify(finalMapping, null, 2);
    }
  
    async function getSheetJSON(file, sheetName) {
      const data = await file.arrayBuffer();
  
      const workbook = XLSX.read(data);
      const targetSheet = workbook.Sheets[sheetName];
  
      const sheetJSON = XLSX.utils.sheet_to_json(targetSheet, {
        raw: true,
        header: 1
      });
  
      return sheetJSON;
    }
  
    function verticalFiller(sheetJSON, rowIndex, holeyArray) {
      let values = [];
  
      for (let index = 0; index < holeyArray.length; index++) {
        if (!holeyArray[index]) {
          let mappedValue;
          while (rowIndex > 0) {
            mappedValue = sheetJSON[rowIndex][index];
            rowIndex--;
            if (mappedValue) {
              break;
            }
          }
          values.push(mappedValue);
        } else {
          values.push(holeyArray[index]);
        }
      }
      return values;
    }
  
    function nestedFiller(source = {}, path = [], value = "") {
      // console.log({ source: JSON.parse(JSON.stringify(source)), path, value });
      let target = source;
      path.forEach((part, index) => {
        if (!target[part]) {
          target[part] = {};
        }
        if (index !== path.length - 1) {
          target = target[part];
        }
      });
      target[path[path.length - 1]] = value;
      return source;
    }
  
    function generateRoleFeatureMapping(
      sheetJSON,
      rowIndex,
      roleColumnStartIndex,
      roleColumnEndIndex,
      mapping = {}
    ) {
      const roleRow = sheetJSON[rowIndex];
      const roles = roleRow.slice(roleColumnStartIndex, roleColumnEndIndex + 1);
  
      const roleMapping = verticalFiller(sheetJSON, rowIndex, roles);
  
      return nestedFiller(
        mapping,
        roleMapping,
        generateFeatureActionMapping(sheetJSON, rowIndex)
      );
    }
  
    function generateFeatureActionMapping(sheetJSON, rowIndex) {
      const featureRow = sheetJSON[FEATURE_NAMES_ROW_INDEX];
  
      const rangeDetails = calculateFeatureRangeDetails(
        sheetJSON,
        FEATURE_NAMES_ROW_INDEX,
        FEATURE_NAMES_COLUMN_START_INDEX
      );
  
      return Object.fromEntries(
        rangeDetails.map((rangeDetail, index) => {
          return [
            rangeDetail.name,
            generateActionPermissionMapping(
              sheetJSON,
              rowIndex,
              rangeDetail.startIndex,
              rangeDetail.endIndex
            )
          ];
        })
      );
    }
  
    function calculateFeatureRangeDetails(sheetJSON, rowIndex, columnStartIndex) {
      const row = sheetJSON[rowIndex];
      const values = row.slice(columnStartIndex);
  
      let rangeDetails = [];
      for (let index = 0; index < values.length; index++) {
        if (values[index]) {
          rangeDetails.push({
            name: values[index],
            startIndex: columnStartIndex + index
          });
        }
        rangeDetails[rangeDetails.length - 1].endIndex = columnStartIndex + index;
      }
      return rangeDetails;
    }
  
    function generateActionPermissionMapping(
      sheetJSON,
      permissionRowIndex,
      actionColumnStartIndex,
      actionColumnEndIndex
    ) {
      const actionRow = sheetJSON[ACTION_NAMES_ROW_INDEX];
      const actions = actionRow.slice(
        actionColumnStartIndex,
        actionColumnEndIndex + 1
      );
  
      const actionPermissionMap = Object.fromEntries(
        actions.map((actionName, index) =>
          generateActionPermissionEntry(
            sheetJSON,
            permissionRowIndex,
            actionColumnStartIndex + index,
            actionName
          )
        )
      );
      return actionPermissionMap;
    }
  
    function generateActionPermissionEntry(
      sheetJSON,
      permissionRowIndex,
      permissionColumnIndex,
      actionName
    ) {
      const permission = sheetJSON[permissionRowIndex][permissionColumnIndex];
  
      return [actionName, convertPermissionToBool(permission)];
    }
  
    function convertPermissionToBool(permission = "") {
      const parsedPermission = permission.toLowerCase();
      return parsedPermission === "accessible";
    }
  </script>
  
  <style>
    main {
      display: flex;
      flex-direction: column;
      justify-content: center;
      gap: 1rem;
      height: 100vh;
    }
  
    textarea {
      flex-grow: 1;
    }
  </style>
  
  <main>
      <label for="file">Input File</label>
      <input type="file" id="file" on:change={transformToMappingJSON}/>
    <textarea value={rolePermissionMapping}/>
  </main>